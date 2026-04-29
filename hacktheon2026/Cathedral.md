# 🕯️ Write-up: Cathedral of the Last Candle (HackTheon Sejong 2026)

**Category:** Misc / Algo / Linear Algebra  
**Flag:** `hacktheon2026{a9ae89213144b4e8e70da8dbeb6360b73bb5a7c6000dab210fc45682ada8bf7258c94c7d293e2c3faaad8851f72afd4f9883c78679abc4417b794938cb2810d8d32ddc01523eca16}`

---

## 📖 Tổng quan bài toán

Người chơi bị quăng vào một nhà thờ với grid $5 \times 8$ (tổng cộng 40 ô), tượng trưng cho 40 ngọn nến đang cháy ở các mức độ khác nhau (giá trị $1, 2, 0 \pmod 3$). Mục tiêu tối thượng là dập tắt toàn bộ nến (đưa cả 40 ô về $0$) và thực thi lệnh `pray` ngay tại Bàn thờ - Altar `(4,7)` để lấy Flag.

Mỗi khi đứng tại một ô trên bàn cờ, bạn có thể gọi hai thao tác (không có tính giao hoán). Cả hai lệnh này cập nhật giá trị của *ô hiện tại $(a)$* và *ô hàng xóm đang liên kết $(b)$*:

- `ring` $\rightarrow [a+b, \; a+2b] \pmod 3$
- `hush` $\rightarrow [2a+2b, \; 2a+b] \pmod 3$

Rào cản lớn nhất: Bạn chỉ có đúng $300$ Budget (lượt tương tác) cho toàn bộ game. Mới nhìn, ai cũng nghĩ đây là một bài Light-Outs kinh điển kết hợp Constraint Satisfaction. Nhưng sự thật không hề đơn giản như vậy!

---

## 🔬 Cái bẫy Đại số tuyến tính (Linear Algebra Trap)

Nếu chúng ta nhìn hai thao tác `ring` ($R$) và `hush` ($H$) dưới lăng kính của Không gian Vector trên Trường Galois $\mathbb{F}_3$, các ma trận chuyển vị tương ứng sẽ là:

$$R = \begin{pmatrix} 1 & 1 \\ 1 & -1 \end{pmatrix} \implies \det(R) = (1)(-1) - 1(1) = -2 \equiv 1 \pmod 3$$

$$H = \begin{pmatrix} -1 & -1 \\ -1 & 1 \end{pmatrix} \implies \det(H) = (-1)(1) - (-1)(-1) = -2 \equiv 1 \pmod 3$$

Bởi vì cả hai ma trận $R$ và $H$ hoạt động trên các ô liên kết, nên hành động thay đổi trạng thái cả bàn cờ tương đương với việc nhân Vector trạng thái với một Ma trận vuông kích thước khổng lồ. Và vì Định thức (Determinant) của các phép chuyển cục bộ luôn bằng đúng $1 \pmod 3$, **về mặt toán học, bạn KHÔNG BAO GIỜ có thể biến một Vector khác 0 thành Vector toàn 0!**

Đó chính là lý do vì sao tác giả đặt tên challenge là: *"Cathedral of the **Last** Candle"* — sẽ luôn luôn còn lại ít nhất một phần dư (residual) không thể bị dập tắt!

---

## 🧩 Khoảnh khắc "Aha": Node thứ 41 vô hình

Nếu toán học đã cấm tiệt việc dập tắt 40 ngọn nến, thì làm sao ta qua mặt được điều kiện của lệnh `pray`? Câu trả lời nằm ở cấu trúc liên kết ngầm của Grid. 

Chúng mình đã viết một đoạn script nhỏ dùng `hush` để dò tìm (probe) cấu trúc liên kết toàn bản đồ. Và đây là kết quả thu được:
- Cấu trúc bản đồ là một Đồ thị cây có hướng (Directed Tree) trỏ từ các node lá (góc trên-trái) về đúng gốc cây (root) là tọa độ `(4,7)`.
- Gốc `(4,7)` **không liên kết với bất kì ô nào khác trên bàn cờ**.
- Thay vào đó, nó trỏ thẳng ra ngoài Map... đi thẳng vào cái **Bàn thờ (Altar / †)**!

Cú lừa ngoạn mục nằm ở đây: Không gian trạng thái thực sự không phải là 40 ô, mà là **41 Ô**. Ô thứ 41 (Altar) ẩn mình, hoạt động như một cái "thùng rác". Chỉ cần ta sweep toàn bộ 40 ô trên bàn cờ về $0$ và đẩy thành công cái phần dư toán học rác rưởi (algebraic debt) kia vào Altar... Boom! Game over!

## 🚀 Thực thi & Thuật toán Solver BFS Toán học

Tuy nhiên, dọn dẹp các nhánh từ dưới lên trên (Bottom-up) sẽ gặp một vấn đề: Nodes cha rất dễ bị kẹt trong một quỹ đạo $R$-orbit đặc biệt (khiến việc gõ lệnh $R$ liên tục chỉ thay đổi trạng thái theo vòng lặp $(1,1) \to (2,0) \to (2,2) \to (1,0)$ mà vĩnh viễn không bao giờ rớt được số $0$). 

Để xử lý, tụi mình đã xây dựng thuật toán gồm 2 Phase kết hợp Breadth-First Sweep (BFS):

1. **Mapping & Probing (Vẽ Map và Dò Altar):**
   Dò bản đồ để sinh Object Tree cấu trúc. Di chuyển tới `(4,7)`, dùng 1 lệnh `ring` để tính toán chính xác giá trị gốc bí mật của Altar (`new_val - old_val`), sau đó chạy `hush` để undo. Giờ ta đã có đủ dữ kiện 41 Node.

2. **Phase 1: Quét lá về gốc (Dọn 38 ô):**
   Sử dụng BFS trên trạng thái `(Current, Parent, Grandparent)` để sinh chuỗi $R/H$ liên hoàn, dập các node thành 0. Ta quét theo Post-Order (Lá duyệt trước). Điều quan trọng: cố tình **để dành lại** (Skip) ô Root `(4,7)` và **chính xác 1 ô Con của Root** làm điểm gánh (Buffer State).

3. **Phase 2: Ép xung phá vỡ Quỹ đạo:**
   Sang Phase 2, chạy một bộ BFS phức hợp cho 3 Node cuối: Node Con gánh, Node Root, và Node Altar. Vì Node Con (khi kích hoạt) sẽ tiêm các "nhiễu" toán học lên Node Root mà không làm ảnh hưởng tới cái Altar, nó sẽ mạnh tay đập vỡ cái Dead_Lock Orbit ở trên. Kết quả: BFS tính thành công chuỗi thao tác đè cả Node Con lẫn Node Root về $0$ trong một nốt nhạc, đẩy toàn bộ rác vào ô Altar.

4. **The Final Prayer (Cầu nguyện):**
   Lập tức dập tắt thành công vĩnh viễn 40/40 ngọn nến. Rảo bước tới `(4,7)` và gõ lệnh `pray`. Với cost budget được tối ưu hoàn hảo, server xuất Flag trong sự câm nín của nhà thờ.

---

## 🛠️ The Python 3 Script

*Script pwntools bypass full game có thể được xem tham khảo tại file `cathedral_solver.py`.*
