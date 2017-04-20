# Nguồn gốc

Do nhu cầu xác thực tập trung, tránh việc người dùng phải ghi nhớ hàng trăm username/password cho hàng trăm dịch vụ. Các công ty bắt đầu triển khai SSO hoặc tích hợp SSO vào 
hệ thống đang triển khai.

Khi bạn gõ "SSO là gì" vào ô tìm kiếm của google thì hiện ra vô số kết quả, đều định nghĩa SSO là gì, làm gì, hoạt động ra sao? Nhưng chả có cái hướng dẫn ra hồn nào để 
bạn dựng được một hệ thống SSO. Vì thực tế SSO có rất nhiều cách triển khai. Đầu tiên bạn phải hiểu cách dựng một hệ thống SSO, các thành phần bên trong, cơ chế giao tiếp, 
xác thực,...

# Cấu trúc

Một hệ thống Web-based Single Sign-On sẽ có 03 phần:
- Principal (typically a user)
- Service Provider (SP)
- Identity Provider (IdP)

Principal ở đây có thể là người dùng hoặc một thực thể, một module nào đó muốn đăng nhập vào hệ thống.

Service Provider tiếp nhận yêu cầu đăng nhập từ Principal, chuyển hướng người dùng sang một phiên đăng nhập do Identity Provider cung cấp, sau khi đăng nhập thành công, IdP 
sẽ gửi lại một identity assertion để SP lưu giữ cho phiên làm việc của người dùng.

Identity Provider có nhiệm vụ xác thực thông tin đăng nhập và sinh ra một identity assertion gửi về cho SP.

Bước đầu tìm hiểu SSO, tôi ứng dụng SSO cho xác thực người dùng trong OpenStack luôn.

# SSO vào OpenStack

Trong phần này, chúng ta cấu hình hệ thống Web-based Single Sign-On gồm OpenStack Keystone, Shibboleth Identity Provider and OpenStack Horizon

# Tham khảo

- [https://xuctarine.blogspot.com/2016/02/keystone-service-provider-with.html](https://xuctarine.blogspot.com/2016/02/keystone-service-provider-with.html)

