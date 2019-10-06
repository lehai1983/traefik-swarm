# traefik-swarm
traefik-swarm
Hướng dẫn này giải thích cách sử dụng Træfik trong chế độ khả dụng cao trong Docker Swarm và với Let Encrypt.

Tại sao chúng ta cần Træfik trong chế độ cụm? Chạy nhiều trường hợp nên làm việc ra khỏi hộp?

Nếu bạn muốn sử dụng Let Encrypt với Træfik, chia sẻ cấu hình hoặc chứng chỉ TLS giữa nhiều phiên bản Træfik, bạn cần cụm Træfik / HA.

Ok, chúng ta có thể gắn một khối lượng chia sẻ được sử dụng bởi tất cả các phiên bản của tôi không? Vâng, bạn có thể, nhưng nó sẽ không hoạt động. Khi bạn sử dụng Let Encrypt, bạn cần lưu trữ chứng chỉ, nhưng không chỉ. Khi Træfik tạo chứng chỉ mới, nó sẽ cấu hình một thách thức và một khi Let Encrypt sẽ xác minh quyền sở hữu tên miền, nó sẽ ping lại thử thách đó. Nếu các trường hợp Træfik khác không biết thách thức, việc xác thực sẽ thất bại.

Để biết thêm thông tin về thử thách: Môi trường quản lý chứng chỉ tự động (ACME)

Điều kiện tiên quyết 
Bạn sẽ cần một cụm Docker Swarm hoạt động.

Cấu hình Træfik 
Trong hướng dẫn này, chúng tôi sẽ không sử dụng tệp cấu hình TOML mà chỉ sử dụng cờ dòng lệnh. Cùng với đó, chúng ta có thể sử dụng hình ảnh cơ sở mà không cần gắn tệp cấu hình hoặc xây dựng hình ảnh tùy chỉnh.

Trfik nên làm gì:

Nghe 80 và 443
Chuyển hướng lưu lượng HTTP sang HTTPS
Tạo chứng chỉ SSL khi tên miền được thêm vào
Nghe sự kiện Docker Swarm
Cấu hình entrypoints 
TL; DR:


$ traefik \
    --entrypoints='Name:http Address::80 Redirect.EntryPoint:https' \
    --entrypoints='Name:https Address::443 TLS' \
    --defaultentrypoints=http,https
Để nghe các cổng khác nhau, chúng ta cần tạo một điểm vào cho mỗi cổng.

Cú pháp CLI là --entrypoints='Name:a_name Address:an_ip_or_empty:a_port options'. Nếu bạn muốn chuyển hướng lưu lượng truy cập từ một điểm nhập cảnh khác, đó là tùy chọn Redirect.EntryPoint:entrypoint_name.

Theo mặc định, chúng tôi không muốn định cấu hình tất cả các dịch vụ của mình để nghe trên http và https, chúng tôi thêm cấu hình điểm nhập mặc định : --defaultentrypoints=http,https.

Chúng ta hãy Mã hóa cấu hình 
TL; DR:


$ traefik \
    --acme \
    --acme.storage=/etc/traefik/acme/acme.json \
    --acme.entryPoint=https \
    --acme.httpChallenge.entryPoint=http \
    --acme.email=contact@mydomain.ca
Let Encrypt cần 4 tham số: điểm nhập TLS để nghe, điểm nhập không phải TLS để cho phép thử thách HTTP, lưu trữ chứng chỉ và email để đăng ký.

Để bật hỗ trợ Let Encrypt, bạn cần thêm --acmecờ.

Bây giờ, Træfik cần biết nơi lưu trữ chứng chỉ, chúng ta có thể chọn giữa một khóa trong kho Lưu trữ khóa-Giá trị hoặc đường dẫn tệp: --acme.storage=my/keyhoặc --acme.storage=/path/to/acme.json.

Các acme.httpChallenge.entryPointlá cờ cho phép HTTP-01thách thức và xác định EntryPoint để sử dụng trong những thách thức.

Đối với email của bạn và điểm vào, nó --acme.entryPointvà --acme.emailcờ.

Cấu hình Docker ¶
TL; DR:


$ traefik \
    --docker \
    --docker.swarmmode \
    --docker.domain=mydomain.ca \
    --docker.watch
Để bật hỗ trợ docker và swarm-mode, bạn cần thêm --dockervà gắn --docker.swarmmodecờ. Để xem các sự kiện docker, thêm --docker.watch.
