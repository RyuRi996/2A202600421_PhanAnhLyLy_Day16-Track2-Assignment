# Hướng dẫn Thực hành LAB 16: Cloud AI Environment Setup (2.5h)

Chào mừng các bạn đến với Lab 16. Trong bài thực hành này, chúng ta sẽ thiết lập một môi trường Cloud AI hoàn chỉnh trên AWS bằng cách sử dụng **Terraform** (Infrastructure as Code) và **Docker/vLLM**. 

Mục tiêu của bài lab là triển khai một mô hình ngôn ngữ lớn (LLM - cụ thể là `google/gemma-4-E2B-it`) lên một máy chủ GPU (NVIDIA T4) nằm an toàn trong mạng nội bộ (Private VPC), và cung cấp API truy cập ra bên ngoài thông qua Load Balancer.

---

## Phần 1: Chuẩn bị tài khoản AWS và thiết lập IAM (Least-Privilege)

Để làm việc với AWS an toàn, chúng ta không bao giờ sử dụng tài khoản Root. Thay vào đó, bạn sẽ tạo một IAM User thuộc một IAM Group với các quyền vừa đủ (least-privilege) để Terraform có thể triển khai hạ tầng.

### Bước 1.1: Truy cập AWS Console
1. Đăng nhập vào [AWS Management Console](https://console.aws.amazon.com/) bằng tài khoản Root hoặc tài khoản Admin của bạn.
2. Trên thanh tìm kiếm, gõ **IAM** và chọn dịch vụ **IAM (Identity and Access Management)**.

### Bước 1.2: Tạo IAM Group và gắn quyền (Policies)
1. Trong menu bên trái của IAM, chọn **User groups** -> click **Create group**.
2. Đặt tên nhóm: `AI-Lab-Group`.
3. Trong phần **Attach permissions policies**, bạn cần tìm và tick chọn các quyền (roles) sau. **Giải thích tại sao cần:**
   - `AmazonEC2FullAccess`: Cần thiết để Terraform tạo máy chủ ảo (Bastion Host, GPU Node), Key Pairs, và Security Groups.
   - `AmazonVPCFullAccess`: Cần thiết để Terraform tạo môi trường mạng (VPC, Subnets, Internet Gateway, NAT Gateway, Route Tables).
   - `ElasticLoadBalancingFullAccess`: Cần thiết để tạo Application Load Balancer (ALB) giúp phân phối traffic từ internet vào private GPU Node.
   - `IAMFullAccess`: Bắt buộc vì Terraform script của chúng ta sẽ tạo một IAM Role và Instance Profile (gắn vào GPU node để cấp quyền cho node nếu cần tương tác với AWS services sau này).
4. Click **Create user group**.

### Bước 1.3: Tạo IAM User và lấy Access Keys
1. Trong menu bên trái, chọn **Users** -> click **Create user**.
2. Đặt tên user: `ai-lab-user`. Click Next.
3. Chọn **Add user to group**, tick chọn nhóm `AI-Lab-Group` vừa tạo. Click Next -> **Create user**.
4. Bấm vào tên user `ai-lab-user` vừa tạo. Chuyển sang tab **Security credentials**.
5. Kéo xuống phần **Access keys**, click **Create access key**.
6. Chọn **Command Line Interface (CLI)** -> Check đồng ý -> Next -> **Create access key**.
7. **LƯU Ý:** Copy `Access key ID` và `Secret access key` lưu vào nơi an toàn. Bạn sẽ không thể xem lại Secret key sau khi đóng cửa sổ này.

### Bước 1.4: Tăng hạn mức vCPU cho GPU (Rất quan trọng)
Theo mặc định, AWS khóa hạn mức sử dụng máy chủ GPU của các tài khoản mới ở mức 0 vCPU để bảo mật. Bạn cần mở khóa để chạy được instance `g4dn.xlarge` (cần 4 vCPU).
1. Trên thanh tìm kiếm của AWS Console, gõ **Service Quotas** và chọn nó.
2. Menu trái chọn **AWS services** -> tìm và chọn **Amazon Elastic Compute Cloud (Amazon EC2)**.
3. Ở ô tìm kiếm của Quotas, gõ `Running On-Demand G and VT instances`.
4. Chọn nó và click **Request quota increase**.
5. Nhập số **4** (tương đương 4 vCPU cho 1 máy `g4dn.xlarge`). 
*Lưu ý: AWS có thể mất từ vài phút đến vài giờ để duyệt yêu cầu này.*

---

## Phần 2: Cài đặt và cấu hình môi trường Local

Trên máy tính cá nhân của bạn, mở Terminal/Command Prompt.

### Bước 2.1: Cấu hình AWS CLI
Đảm bảo bạn đã cài đặt [AWS CLI](https://aws.amazon.com/cli/). Gõ lệnh sau để cấu hình tài khoản vừa tạo:
```bash
aws configure
```
Nhập các thông tin:
- **AWS Access Key ID**: (Dán Access key ID của bạn)
- **AWS Secret Access Key**: (Dán Secret access key của bạn)
- **Default region name**: `us-east-1` (Bắt buộc dùng us-east-1 cho lab này)
- **Default output format**: `json`

### Bước 2.2: Lấy Hugging Face Token
Mô hình `google/gemma-4-E2B-it` là một mô hình bị giới hạn (gated model). Bạn cần cấp quyền truy cập cho Terraform.
1. Đăng nhập [Hugging Face](https://huggingface.co/).
2. Vào trang của model [google/gemma-4-E2B-it](https://huggingface.co/google/gemma-4-E2B-it) và đồng ý với điều khoản (Accept license).
3. Vào **Settings** -> **Access Tokens** -> Tạo một token (quyền Read) và copy lại.

---

## Phần 3: Triển khai Hạ tầng với Terraform

Terraform là công cụ giúp chúng ta khởi tạo hạ tầng AWS hoàn toàn tự động bằng code. Kiến trúc bao gồm:
- Mạng **Private VPC** cách ly hoàn toàn với bên ngoài.
- **Bastion Host** (t3.micro) ở Public Subnet: Dùng làm trạm trung chuyển an toàn nếu cần SSH vào GPU Node.
- **GPU Node** (g4dn.xlarge - T4 GPU) ở Private Subnet: Chạy Docker chứa vLLM để load model AI.
- **NAT Gateway**: Cho phép Private Subnet kéo image Docker và tải Model từ internet.
- **Application Load Balancer (ALB)**: Mở cổng 80 (HTTP) để nhận API request và đẩy vào GPU node ở cổng 8000.

### Bước 3.1: Khởi tạo Terraform
Di chuyển vào thư mục code Terraform:
```bash
cd terraform
terraform init
```

### Bước 3.2: Cấu hình biến môi trường
Thiết lập Token Hugging Face của bạn để Terraform truyền vào máy chủ EC2 khi khởi động:
```bash
export TF_VAR_hf_token="<DÁN_TOKEN_HUGGING_FACE_CỦA_BẠN_VÀO_ĐÂY>"
```

### Bước 3.3: Triển khai (Apply)
Chạy lệnh apply để Terraform bắt đầu tạo tài nguyên trên AWS:
```bash
terraform apply
```
Gõ `yes` khi được hỏi. Quá trình này sẽ mất khoảng **10 đến 15 phút** (phần lớn thời gian là để khởi tạo NAT Gateway).

*Mẹo: Các bạn hãy bắt đầu bấm giờ (benchmark) từ lúc gõ `yes` ở bước này nhé!*

---

## Phần 4: Kiểm tra AI Endpoint (Inference)

Khi `terraform apply` chạy xong, màn hình terminal sẽ in ra các thông số quan trọng (Outputs). Trông sẽ giống thế này:
```text
Outputs:

alb_dns_name = "ai-inference-alb-xxxxxx.us-east-1.elb.amazonaws.com"
bastion_public_ip = "100.x.x.x"
endpoint_url = "http://ai-inference-alb-xxxxxx.us-east-1.elb.amazonaws.com/v1/completions"
gpu_private_ip = "10.0.1x.x"
```

**Quan trọng:** Mặc dù Terraform đã báo thành công, GPU Node vẫn đang ngầm tải Docker image (vLLM) và model weights (~vài GB) từ Hugging Face. **Bạn cần đợi thêm 5-10 phút** để model sẵn sàng.

### Bước 4.1: Gọi API bằng cURL
Thay thế URL của ALB bạn nhận được vào lệnh dưới đây và chạy thử:

```bash
curl -X POST http://<THAY_BẰNG_ALB_DNS_NAME_CỦA_BẠN>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-E2B-it",
    "messages": [
      {"role": "system", "content": "Bạn là một trợ lý AI hữu ích."},
      {"role": "user", "content": "Hãy giải thích Bastion Host trong AWS là gì?"}
    ],
    "max_tokens": 150
  }'
```
Nếu nhận được câu trả lời từ AI, chúc mừng bạn đã triển khai thành công! Hãy ghi lại tổng thời gian (Cold start time) từ lúc chạy `terraform apply` đến lúc nhận được API response đầu tiên.

---

## Phần 5: Tiêu chí nộp bài (Deliverables)

Để hoàn thành Lab 16, sinh viên cần thu thập và nộp các kết quả sau:
1. **Ảnh chụp màn hình (Screenshot) API gọi thành công:** Chụp lại lệnh curl và câu trả lời của AI.
2. **Ảnh chụp màn hình AWS Billing/Cost Dashboard:** 
   - Vào AWS Console -> Gõ **Billing** trên thanh tìm kiếm.
   - Chụp lại màn hình thể hiện các dịch vụ đang chạy phát sinh chi phí (EC2, NAT Gateway).
3. **Report Cold Start Time:** Ghi lại tổng thời gian triển khai (Mục tiêu: < 15 phút cho instance T4).
4. **Mã nguồn:** Nén thư mục chứa file Terraform đã chạy thành công.

---

## Phần 6: Dọn dẹp tài nguyên (CỰC KỲ QUAN TRỌNG)

GPU EC2 (`g4dn.xlarge`) và NAT Gateway tính phí theo giờ và **rất đắt**. Ngay sau khi test thành công và chụp ảnh nộp bài, bạn **BẮT BUỘC** phải xóa toàn bộ tài nguyên để tránh mất tiền.

Chạy lệnh sau trong thư mục `terraform`:
```bash
terraform destroy
```
Gõ `yes` khi được hỏi. Quá trình xóa sẽ mất khoảng 5 phút. Hãy đợi đến khi terminal báo `Destroy complete!` để chắc chắn mọi thứ đã bị xóa.