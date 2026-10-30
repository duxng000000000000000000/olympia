# Đường đua tri thức

Web game thi kiến thức phong cách gameshow, tối ưu cho một máy tính điều khiển + máy chiếu và 5 điện thoại thí sinh.

## Tính năng
- Màn MC: tạo phòng, đổi tên 5 thí sinh, quản lý bảng điểm và ngân hàng câu hỏi.
- Màn chiếu: hiển thị câu hỏi lớn, đồng hồ đếm ngược và bảng điểm 5 thí sinh.
- Điện thoại: tham gia bằng mã phòng/QR và gửi đáp án trong phần Tăng tốc.
- Tăng tốc: MC chọn câu hỏi, 5 thí sinh gửi đáp án đồng thời; MC đánh dấu đúng/sai. Đáp án xếp theo thời điểm máy chủ nhận: 40 / 30 / 20 / 10 / 0. Đáp án sai nhận 0 điểm.
- Về đích: tự xếp lượt theo điểm từ cao xuống thấp; mỗi thí sinh có 2 câu; MC chọn gói 10 / 20 / 30 điểm và quyết định đúng/sai.
- Ngôi sao hy vọng: tối đa một lần mỗi thí sinh; đáp án đúng nhân đôi điểm, sai bị trừ gấp đôi.
- Giành quyền: khi thí sinh chính trả lời sai, MC có thể bấm tên thí sinh khác hoặc nhấn phím 1–5 để xác định người giành quyền. MC chấm đúng/sai.
- Có hai kho câu hỏi độc lập: Tăng tốc (câu hỏi + đáp án, không có gói điểm) và Về đích (mỗi câu thuộc gói 10/20/30). Mỗi kho có chức năng thêm, xóa, nhập và xuất JSON riêng. Có sẵn câu hỏi mẫu cho cả hai phần.

## Chạy thử
Mở index.html bằng trình duyệt. Khi chưa cấu hình Firebase, giao diện MC và màn chiếu có chế độ demo lưu trong trình duyệt. Chế độ demo không đồng bộ giữa các thiết bị khác nhau.

## Đưa lên GitHub Pages
1. Vào repository Settings → Pages.
2. Ở phần Build and deployment, chọn Deploy from a branch.
3. Chọn branch main, folder /(root), nhấn Save.
4. Chờ GitHub Pages xuất bản, sau đó mở URL được cung cấp.

- MC: https://USERNAME.github.io/olympia/
- Màn chiếu: thêm ?mode=screen&room=MA_PHONG
- Điện thoại: thêm ?mode=player&room=MA_PHONG

## Bật kết nối 5 điện thoại (Firebase Realtime Database)
GitHub Pages là hosting tĩnh; để 5 thiết bị trao đổi dữ liệu trực tiếp, cần một backend realtime. Ứng dụng dùng Firebase Authentication (Anonymous) và Realtime Database.

1. Tạo project tại https://console.firebase.google.com/.
2. Thêm ứng dụng Web trong Project Overview → Add app.
3. Vào Authentication → Sign-in method, bật Anonymous.
4. Tạo Realtime Database và chọn location phù hợp.
5. Sao chép firebase-config.js.example thành firebase-config.js, rồi điền thông số Web App từ Firebase:

    window.FIREBASE_CONFIG = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT.firebaseapp.com",
      databaseURL: "https://YOUR_PROJECT-default-rtdb.asia-southeast1.firebasedatabase.app",
      projectId: "YOUR_PROJECT",
      appId: "YOUR_APP_ID"
    };

6. Đặt firebase-config.js ở cùng thư mục với index.html và deploy lại GitHub Pages.
7. Trong Realtime Database → Rules, dùng rules mẫu dưới đây rồi nhấn Publish:

    {
      "rules": {
        "rooms": {
          "$room": {
            ".read": "auth != null",
            ".write": "auth != null && ((!data.exists() && newData.child('hostUid').val() === auth.uid) || data.child('hostUid').val() === auth.uid)",
            "players": {
              "$uid": {
                ".write": "auth != null && auth.uid === $uid && root.child('rooms').child($room).child('state').exists()",
                ".validate": "newData.hasChildren(['slot', 'name']) && newData.child('slot').isNumber() && newData.child('slot').val() >= 1 && newData.child('slot').val() <= 5"
              }
            },
            "buzz": {
              "$question": {
                ".read": "auth != null",
                ".write": "auth != null && !data.exists() && root.child('rooms').child($room).child('players').child(auth.uid).exists() && root.child('rooms').child($room).child('players').child(auth.uid).child('slot').val() === newData.child('slot').val() && newData.child('uid').val() === auth.uid && root.child('rooms').child($room).child('state').child('phase').val() === 'final' && root.child('rooms').child($room).child('state').child('final').child('current').child('status').val() === 'steal' && root.child('rooms').child($room).child('state').child('final').child('current').child('qid').val() === $question && root.child('rooms').child($room).child('state').child('final').child('current').child('slot').val() !== newData.child('slot').val()",
                ".validate": "newData.hasChildren(['slot', 'uid', 'name', 'claimedAt']) && newData.child('slot').isNumber() && newData.child('claimedAt').isNumber()"
              }
            },
            "submissions": {
              "boost": {
                "$question": {
                  "$uid": {
                    ".write": "auth != null && auth.uid === $uid && root.child('rooms').child($room).child('players').child(auth.uid).exists() && root.child('rooms').child($room).child('state').child('boost').child('active').val() === true && root.child('rooms').child($room).child('state').child('boost').child('id').val() === $question && !data.exists()",
                    ".validate": "newData.hasChildren(['slot', 'answer', 'submittedAt'])"
                  }
                }
              }
            }
          }
        }
      }
    }

Firebase configuration in a browser is not a private secret. Database Rules are the important access-control layer. These sample rules reserve state changes to the room creator and allow each authenticated contestant account one submission per speed-round question. Test with a private room before the event; do not enable public read/write access to the whole database.

## Luật và cách chạy
1. MC mở trang chính, bấm Tạo phòng. Sau khi kết nối Firebase, QR và link tham gia sẽ xuất hiện.
2. Chiếu URL màn hình với query ?mode=screen&room=... lên projector.
3. Năm thí sinh quét QR, chọn số báo danh 1–5 và nhập tên.
4. MC nhấn Bắt đầu Tăng tốc, lần lượt chọn câu hỏi để chiếu. Sau khi mọi người gửi hoặc hết giờ, khóa câu trả lời, chấm đúng/sai rồi chấm điểm.
5. MC chuyển sang Về đích. Chọn Ngôi sao hy vọng (nếu thí sinh muốn dùng) rồi chọn gói điểm 10/20/30. MC đọc câu hỏi và tự chấm.
6. Nếu người chính sai, MC mở quyền giành chuông cho các thí sinh còn lại. Có thể nhấn nút tên thí sinh hoặc phím số 1–5.
7. Mỗi thí sinh trả lời hai câu Về đích. Hết năm lượt, chương trình hiện bảng điểm chung cuộc.

## Định dạng ngân hàng câu hỏi JSON

**Tăng tốc** (không có trường `points`):

    [
      {
        "id": "boost-001",
        "topic": "Toán học",
        "q": "Câu hỏi viết ở đây?",
        "a": "Đáp án dành cho MC"
      }
    ]

**Về đích** (bắt buộc `points` là 10, 20 hoặc 30):

    [
      {
        "id": "final-001",
        "points": 20,
        "topic": "Toán học",
        "q": "Câu hỏi viết ở đây?",
        "a": "Đáp án dành cho MC"
      }
    ]

Trường `a` là đáp án tham khảo để MC đọc, không tự động quyết định đúng/sai. Các câu hỏi đã có trong kho cũ được tự động giữ lại ở kho Về đích.