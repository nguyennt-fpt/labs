---
title: "ECS Fargate Microservices"
date: "2025-08-09"
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

Dự án này triển khai một **nền tảng triển khai microservices tự động và bảo mật hoàn toàn** trên AWS, sử dụng **ECS Fargate** làm công cụ điều phối container và **Terraform** làm công cụ Infrastructure-as-Code (IaC).
Mục tiêu là xây dựng một **kiến trúc sẵn sàng cho production** có khả năng **mở rộng, tối ưu chi phí và bảo mật cao**, đồng thời cho phép phân phối ứng dụng nhanh chóng và đáng tin cậy thông qua **pipeline CI/CD** hoàn chỉnh.

Trung tâm của hệ thống là **dịch vụ ECS Fargate** chạy các microservices được container hóa. Tất cả các thành phần hạ tầng AWS—bao gồm mạng, bảo mật, tính toán, lưu trữ và giám sát—được cung cấp thông qua **các module Terraform**. Dự án tích hợp **AWS CodeBuild** và **AWS CodePipeline**, cho phép các bản cập nhật ứng dụng được tự động build, container hóa, đẩy lên **Amazon ECR**, và triển khai lên ECS bất cứ khi nào có code mới được push lên **GitHub**.

Để cân bằng giữa bảo mật và chi phí, hệ thống hoạt động trong **private subnets không có NAT Gateway**. Thay vào đó, nó sử dụng một tập hợp các **VPC Interface Endpoints** (ECR, S3, Secrets Manager, CloudWatch, v.v.) để cho phép các ECS tasks kéo images một cách bảo mật và truy cập các dịch vụ AWS cần thiết mà không cần phơi bày tài nguyên ra internet công cộng.

**Bảo mật được tăng cường ở nhiều lớp:**

- **AWS WAF** bảo vệ Application Load Balancer (ALB) khỏi các cuộc tấn công web phổ biến.

- **Amazon GuardDuty**, **VPC Flow Logs**, và **CloudWatch** cung cấp phát hiện mối đe dọa liên tục và khả năng quan sát.

- **Secrets Manager** được sử dụng để lưu trữ các thông tin nhạy cảm như mật khẩu cơ sở dữ liệu.

- **Route 53** cung cấp quản lý domain tùy chỉnh và định tuyến DNS.

Dự án này thể hiện một **cách tiếp cận hiện đại, cloud-native và phù hợp với production** để chạy microservices trên AWS—kết hợp **tự động hóa, khả năng quan sát, tính khả dụng cao và các thực hành bảo mật mạnh mẽ**.

Đây là kiến trúc của dự án:

![Sơ đồ Kiến trúc](/images/1-ECS-Fargate-Microservices/1.1-Introduction/architecture.drawio.png)