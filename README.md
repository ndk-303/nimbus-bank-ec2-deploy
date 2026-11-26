# Triển khai EC2 bằng Docker Hub

Hướng dẫn này chạy **backend Node.js và Face API** bằng image đã build trên máy phát triển. EC2 pull image từ Docker Hub, không build source trên máy 1 GB RAM. Firebase, S3, FPT.AI và Stripe vẫn là dịch vụ bên ngoài. Ví dụ dùng Ubuntu, Docker Compose, Nginx trên host và domain `api.example.com`.

Repository này chỉ chứa cấu hình triển khai, không chứa source backend/Face API, Docker images, credentials hoặc model. Clone repository vào `/srv/nimbus` trên EC2. Các image phải được build từ repository source `nimbus-bank-android` và push lên Docker Hub trước.

Image mặc định cho EC2 x86_64 được build từ source commit `46db7981a485`: `khoadev303/nimbus-backend:46db7981a485` và `khoadev303/nimbus-face-api:46db7981a485`. Khi cập nhật source, build và push tag mới theo mục 2, sau đó đổi image tag trên EC2.

Máy 1 GB RAM + 2 GB swap chỉ phù hợp thử nghiệm tải thấp; chưa có kết quả đo chứng minh đủ bộ nhớ cho model và traffic của bạn. Theo dõi RAM, swap và tình trạng container khi chạy thực tế.

## 1. Chuẩn bị

- Tài khoản Docker Hub và hai repository: `<user>/nimbus-backend`, `<user>/nimbus-face-api`. Chọn private nếu không muốn công khai images. Image chứa source/ứng dụng nhưng không chứa key hay model.
- EC2 Ubuntu, public IP, domain trỏ đến IP; inbound TCP 80/443, SSH 22 chỉ từ IP quản trị. Không mở 3001 hoặc 8000.
- Docker Engine và Compose trên máy build lẫn EC2; cài EC2 theo [Docker Docs](https://docs.docker.com/engine/install/ubuntu/). Nginx và Certbot chạy trên EC2.
- Trên EC2, chạy `uname -m`: `x86_64` dùng `linux/amd64`; `aarch64` dùng `linux/arm64`. Kiểm tra build Face API và các dependency cho kiến trúc đã chọn; không giả định image AMD64 chạy native trên ARM.
- Firebase project test, Admin service-account JSON, bucket S3 có versioning, FPT.AI key, hai model ONNX và `FACE_TEMPLATE_KEY_HEX` nếu dùng eKYC. Key backend phải được cung cấp trên server, không thêm vào Git hoặc Docker image.

Nếu EC2 chưa có swap, cấu hình 2 GB swap trước khi chạy; không tạo đè swapfile đang dùng. Kiểm tra `free -h`, `swapon --show` và `df -h` trước/sau. Chừa đủ dung lượng đĩa cho image, model, log và swap.

## 2. Build và push từ máy phát triển

Đảm bảo bạn đã đăng nhập Docker Hub bằng `docker login` trên máy phát triển. Từ root của repository **source `nimbus-bank-android`**, không phải repository triển khai này, thay `<user>`, `<tag>` và `linux/amd64` theo tài khoản, phiên bản và kiến trúc EC2. Nên dùng tag cố định, ví dụ commit hash, thay vì chỉ dùng `latest`.

```powershell
docker buildx build --platform linux/amd64 --push -t <user>/nimbus-backend:<tag> ./backend
docker buildx build --platform linux/amd64 --push -t <user>/nimbus-face-api:<tag> ./face-api
```

Lệnh `--push` thực sự upload image lên Docker Hub. Kiểm tra hai repository và tag trên Docker Hub trước khi triển khai. Build trên kiến trúc khác có thể dùng emulation và mất nhiều thời gian; nếu Face API không build được cho ARM, cần kiểm tra các thư viện Python/model cho ARM trước khi chọn EC2. Xem [Docker multi-platform builds](https://docs.docker.com/build/building/multi-platform/).

## 3. Cấu hình trên EC2

Clone repository triển khai này vào `/srv/nimbus` trên EC2. Từ root của bản sao trên EC2:

```sh
cp deploy/ec2.env.example deploy/.env.ec2
nano deploy/.env.ec2
```

Điền hai image **đúng tag đã push**, đường dẫn tuyệt đối tới Firebase JSON và thư mục model:

```dotenv
NIMBUS_BACKEND_IMAGE=<user>/nimbus-backend:<tag>
NIMBUS_FACE_IMAGE=<user>/nimbus-face-api:<tag>
FIREBASE_ADMIN_KEY_PATH=/srv/nimbus/secrets/firebase-admin.json
FACE_MODEL_PATH=/srv/nimbus/private-face-model
```

File `deploy/.env.ec2` dùng cho biến Compose và bị Git ignore; nó không thay thế `backend/.env`. Đặt model tại:

```text
/srv/nimbus/private-face-model/models/buffalo_l/det_10g.onnx
/srv/nimbus/private-face-model/models/buffalo_l/w600k_r50.onnx
```

Face API yêu cầu chính xác hai file ONNX trên trong thư mục `buffalo_l`; xem quyền sử dụng [InsightFace Model Zoo](https://github.com/deepinsight/insightface/tree/master/model_zoo). Đảm bảo container user UID `65532` đọc được file model và thư mục. Backend chạy UID `10001`; file Firebase JSON phải cho user đó đọc được khi bind mount, nhưng không nên cấp quyền đọc công khai. Cấp quyền phù hợp với cách quản lý user/group trên EC2 và kiểm tra sau khi chạy.

Tạo `backend/.env` trên EC2 bằng cách sao chép `backend/.env.example`, sau đó điền các giá trị thực:

```sh
cp backend/.env.example backend/.env
nano backend/.env
```

Các biến chính:

```dotenv
APP_MODE=connected
FIREBASE_PROJECT_ID=<firebase-test-project-id>
OTP_SECRET=<secret-rieng-it-nhat-32-ky-tu>
FPT_AI_KEY=<key-neu-dung-ocr>
OPENROUTESERVICE_API_KEY=<key-neu-dung-chi-duong>
AWS_REGION=<region>
S3_BUCKET=<bucket-test-co-versioning>
FACE_TEMPLATE_KEY_HEX=<64-ky-tu-hex>
# STRIPE_SECRET_KEY=sk_test_...
# STRIPE_WEBHOOK_SECRET=whsec_...
```

Để trống hoặc bỏ các biến không dùng, không giữ placeholder. Không đặt `FIREBASE_AUTH_EMULATOR_HOST` hoặc `FIRESTORE_EMULATOR_HOST`. Giữ `FACE_TEMPLATE_KEY_HEX` ổn định khi dùng lại các template đã tạo. `HOST`, `PORT`, `FACE_API_BASE_URL` và `GOOGLE_APPLICATION_CREDENTIALS` được Compose đặt trong container. Giới hạn quyền đọc file env; không in env hoặc key vào log.

Đối với S3, gắn IAM role có quyền tối thiểu cho EC2 thay vì nhúng access key dài hạn. Backend dùng AWS SDK credential chain; kiểm tra container thực sự truy cập được IMDSv2 và role. AWS lưu ý hop limit `1` có thể gây lỗi trong môi trường container; chỉ điều chỉnh sau khi xem xét quyền truy cập metadata trên máy. [AWS: EC2 instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html)

## 4. Pull và chạy

Nếu images private, đăng nhập Docker Hub trên EC2 với tài khoản có quyền pull. Tại root bản sao trên EC2:

```sh
docker compose --env-file deploy/.env.ec2 -f deploy/compose.ec2.yaml config --quiet
docker compose --env-file deploy/.env.ec2 -f deploy/compose.ec2.yaml pull
docker compose --env-file deploy/.env.ec2 -f deploy/compose.ec2.yaml up -d --no-build
docker compose --env-file deploy/.env.ec2 -f deploy/compose.ec2.yaml ps
curl --fail http://127.0.0.1:3001/health
```

Backend chỉ mở cổng `3001` trên loopback của EC2. Face API chỉ có mặt trong mạng Docker nội bộ. Xem log và xác nhận Face API healthy:

```sh
docker compose --env-file deploy/.env.ec2 -f deploy/compose.ec2.yaml logs --tail 100 backend face-api
docker stats
free -h
```

`/health` backend không chứng minh Firebase, S3, FPT.AI hay Face API hoạt động. Dùng tài khoản thử nghiệm kiểm tra đăng nhập, OCR, selfie và nộp hồ sơ. Ảnh mặt trước được gửi tới FPT.AI; chỉ sử dụng ảnh có quyền xử lý. Luồng hiện tại không đối chiếu selfie với ảnh trên thẻ và không kiểm tra tính thật CCCD.

## 5. Công khai qua HTTPS

Trỏ DNS `api.example.com` về EC2 và đợi phân giải. Copy `deploy/nginx/nimbus-bank.conf.example` vào cấu hình Nginx, thay domain, enable site rồi chạy `sudo nginx -t` trước khi reload. Cấu hình mẫu cho phép request body 6 MB và proxy tới `127.0.0.1:3001` để hỗ trợ ảnh OCR. Cài Certbot theo [hướng dẫn chính thức](https://certbot.eff.org/instructions) và cấp chứng chỉ cho domain thực, sau đó kiểm tra:

```sh
sudo certbot --nginx -d api.example.com --redirect
sudo certbot renew --dry-run
curl --fail https://api.example.com/health
```

Nếu dùng Stripe test, tạo webhook endpoint `https://api.example.com/webhooks/stripe` trên Stripe và đưa signing secret của **endpoint đó** vào `STRIPE_WEBHOOK_SECRET`. Tạo lại backend container sau khi thay env. Không dùng mặc định signing secret của Stripe CLI local cho endpoint này.

Trên máy build Android, đặt `APP_MODE=connected` và `ANDROID_BACKEND_URL=https://api.example.com` trong `.env` root rồi build/cài lại app. `local.properties` không phải nguồn cấu hình backend URL trong Gradle hiện tại.

## 6. Cập nhật và vận hành

Build/push tag mới từ máy phát triển, đổi hai giá trị image trong `deploy/.env.ec2`, rồi chạy lại `pull` và `up -d --no-build` như mục 4. Giữ lại tag cũ để có thể rollback bằng cách đặt lại tag và chạy hai lệnh đó. Không chạy seed hoặc migration tự động khi cập nhật image.

API không tự chạy outbox worker. Nếu sử dụng fulfillment, cần lập lịch chạy worker và theo dõi lỗi; có thể chạy một lần từ root bản sao trên EC2:

```sh
docker compose --env-file deploy/.env.ec2 -f deploy/compose.ec2.yaml run --rm --no-deps backend node dist/scripts/process-outbox.js
```

Worker là tiến trình bổ sung, có thể gây áp lực bộ nhớ trên máy 1 GB; đo trước khi lập lịch. Cấu hình backup, giám sát, giới hạn log và bảo vệ secret phù hợp môi trường của bạn. Không dùng hệ thống demo này cho dữ liệu khách hàng thật.
