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
```
#!/usr/bin/env python3
"""
Cathedral of the Last Candle - PERFECT 41-NODE SOLVER v13
Target: All 40 grid cells = . (value 0).
The Altar is explicitly modeled as the 41st node in the tree.
This guarantees perfect mathematical synchronization.
"""
from pwn import *
import re, sys
from collections import defaultdict, deque

HOST, PORT = '3.35.223.94', 9000
ROWS, COLS = 5, 8
C2V = {'*':1, '~':2, '.':0}

def ring_k(a, b, k):
    for _ in range(k):
        a, b = (a+b)%3, (a+2*b)%3
    return a, b

def parse(out):
    pos = bud = None
    m = re.search(r'Pos:\((\d+),(\d+)\)', out)
    if m: pos = (int(m.group(1)), int(m.group(2)))
    m = re.search(r'Budget:(\d+)', out)
    if m: bud = int(m.group(1))
    cells = {}
    rows = []
    for line in out.split('\n'):
        s = line.strip().strip('║').strip()
        if not s or any(k in s for k in ['CATHEDRAL','Pos:','Budget:','ring','hush',
                                          'Extinguish','Command','a =','b =','═']): continue
        toks = re.findall(r'\[[^\]]*\]|†|[^\s†]+', s)
        if toks and any(t.strip('[]') in ('*','~','.','?') for t in toks if t!='†'):
            rows.append(toks)
    for ri, toks in enumerate(rows):
        ci = 0
        for t in toks:
            if t == '†': continue
            cells[(ri, ci)] = t.strip('[]')
            ci += 1
    return pos, bud, cells

class Solver:
    def __init__(self):
        context.log_level = 'warn'
        self.r = remote(HOST, PORT)
        self.g = {}
        self.pos = (0,0)
        self.bud = 300
        self.bonds = {}
        resp = self.r.recvuntil(b'> ', timeout=10).decode()
        p, b, c = parse(resp)
        if p: self.pos = p
        if b: self.bud = b
        for k,v in c.items():
            if v != '?': self.g[k] = v

    def cmd(self, c):
        self.r.sendline(c.encode())
        resp = self.r.recvuntil(b'> ', timeout=10).decode()
        p, b, cells = parse(resp)
        if p: self.pos = p
        if b is not None: self.bud = b
        for k,v in cells.items():
            if v != '?': self.g[k] = v
        return resp

    def mv(self, d):
        old = self.pos; self.cmd(f'go {d}'); return self.pos != old

    def goto(self, r, c):
        while self.pos[0]<r: self.mv('s')
        while self.pos[0]>r: self.mv('n')
        while self.pos[1]<c: self.mv('e')
        while self.pos[1]>c: self.mv('w')

    def explore_and_discover(self):
        self.goto(0, 0)
        for r in range(ROWS):
            cols = range(COLS) if r%2==0 else range(COLS-1,-1,-1)
            for c in cols:
                if r%2==0:
                    while self.pos[1] < c: self.mv('e')
                else:
                    while self.pos[1] > c: self.mv('w')
                while self.pos[0] < r: self.mv('s')
                before = {}
                for dr,dc in [(-1,0),(1,0),(0,-1),(0,1)]:
                    nr, nc = r+dr, c+dc
                    if 0<=nr<ROWS and 0<=nc<COLS:
                        before[(nr,nc)] = self.g.get((nr,nc))
                self.cmd('hush')
                bond_target = None
                for (nr,nc), old_val in before.items():
                    new_val = self.g.get((nr,nc))
                    if old_val is not None and new_val != old_val:
                        bond_target = (nr, nc)
                        break
                self.cmd('ring')  # undo
                self.bonds[(r,c)] = bond_target
            if r < ROWS-1: self.mv('s')

        roots = [k for k,v in self.bonds.items() if v is None]
        print(f"[*] Bonds discovered. Root(s): {roots} Budget:{self.bud}")
        return roots[0] if roots else None

    def get_altar_val(self):
        """Probe the actual value of the altar (the 41st cell)."""
        self.goto(4, 7)
        old_val = C2V.get(self.g.get((4,7), '~'), 2)
        self.cmd('ring')
        new_val = C2V.get(self.g.get((4,7), '~'), 2)
        b_val = (new_val - old_val) % 3
        self.cmd('hush')
        return b_val

    def pgrid(self):
        for r in range(ROWS):
            print(f" {r}:" + " ".join(self.g.get((r,c),'?') for c in range(COLS)))
        s=sum(1 for v in self.g.values() if v=='*')
        d=sum(1 for v in self.g.values() if v=='.')
        t=sum(1 for v in self.g.values() if v=='~')
        print(f" *={s} .={d} ~={t} bud={self.bud}")

    def simulate_sweep(self, root, altar_initial):
        children = defaultdict(list)
        for c, p in self.bonds.items():
            if p is not None:
                children[p].append(c)

        # Add the ALTAR to our tree representation so math covers 41 nodes.
        self.bonds[root] = 'ALTAR'
        self.bonds['ALTAR'] = None  # Altar has no parent

        buffer_child = children[root][0] if children[root] else None
        
        order = []
        def dfs(node):
            if node == 'ALTAR': return
            for child in children[node]:
                dfs(child)
            order.append(node)
        dfs(root)

        vals = {(r,c): C2V.get(self.g.get((r,c),'~'), 2) for r in range(ROWS) for c in range(COLS)}
        vals['ALTAR'] = altar_initial
        ops = []

        # PHASE 1: Zero out 38 cells
        for u in order:
            if u == root or u == buffer_child:
                continue
                
            p = self.bonds[u]
            gp = self.bonds[p]

            # Since u is NEVER root, p is NEVER ALTAR.
            # Thus gp exists in vals dict.
            Q = deque([(vals[u], vals[p], vals[gp], [])])
            visited = set([(vals[u], vals[p], vals[gp])])
            
            found_seq = []
            while Q:
                u_val, p_val, gp_val, seq = Q.popleft()
                if u_val == 0:
                    found_seq = seq
                    break
                for k in [1, 2, 3]:
                    nu, np = ring_k(u_val, p_val, k)
                    st = (nu, np, gp_val)
                    if st not in visited: visited.add(st); Q.append((nu, np, gp_val, seq + [('u', k)]))
                for k in [1, 2, 3]:
                    np, ngp = ring_k(p_val, gp_val, k)
                    st = (u_val, np, ngp)
                    if st not in visited: visited.add(st); Q.append((u_val, np, ngp, seq + [('p', k)]))
            
            if not found_seq and vals[u] != 0:
                print(f"[!] Warning: Could not find Phase 1 BFS zero seq for {u}")
                
            for target_node_type, k in found_seq:
                if target_node_type == 'u':
                    target = u; na, nb = ring_k(vals[u], vals[p], k); vals[u] = na; vals[p] = nb
                else:
                    target = p; na, nb = ring_k(vals[p], vals[gp], k); vals[p] = na; vals[gp] = nb
                ops.append((target, k))

        # PHASE 2: Zero the buffer_child and the root simultaneously!
        if buffer_child:
            Q = deque([(vals[buffer_child], vals[root], vals['ALTAR'], [])])
            visited = set([(vals[buffer_child], vals[root], vals['ALTAR'])])
            
            found_seq = []
            while Q:
                c_val, r_val, a_val, seq = Q.popleft()
                if c_val == 0 and r_val == 0:
                    found_seq = seq
                    break
                # Ops at buffer_child
                for k in [1, 2, 3]:
                    nc, nr = ring_k(c_val, r_val, k)
                    st = (nc, nr, a_val)
                    if st not in visited: visited.add(st); Q.append((nc, nr, a_val, seq + [('c', k)]))
                # Ops at root
                for k in [1, 2, 3]:
                    nr, na = ring_k(r_val, a_val, k)
                    st = (c_val, nr, na)
                    if st not in visited: visited.add(st); Q.append((c_val, nr, na, seq + [('r', k)]))
            
            if not found_seq and (vals[buffer_child] != 0 or vals[root] != 0):
                print("[!] Warning: Could not find Phase 2 BFS final seq!")
                
            for target_node_type, k in found_seq:
                if target_node_type == 'c':
                    target = buffer_child; nc, nr = ring_k(vals[buffer_child], vals[root], k); vals[buffer_child] = nc; vals[root] = nr
                else:
                    target = root; nr, na = ring_k(vals[root], vals['ALTAR'], k); vals[root] = nr; vals['ALTAR'] = na
                ops.append((target, k))

        non0 = sum(1 for c, v in vals.items() if c != 'ALTAR' and v != 0)
        print(f"[*] Sim finalized: {len(ops)} ops. Grid sum check: {non0} non-. cells (ignoring Altar)")
        val_str = {0:'.', 1:'*', 2:'~'}
        print(f"[*] Simulated Final Root: {val_str[vals[root]]}")
        print(f"[*] Simulated Final Buffer: {val_str[vals[buffer_child]]}")
        return ops

    def execute(self, ops):
        for cell, k in ops:
            if cell == 'ALTAR':
                print("[!] Error: tried to target Altar!")
                continue
            self.goto(*cell)
            if k == 3:
                self.cmd('hush')
            else:
                for _ in range(k):
                    self.cmd('ring')

    def solve(self):
        print("[*] Phase 1 - Mapping Structure...")
        root = self.explore_and_discover()
        if not root: root = (4,7)
        
        altar_val = self.get_altar_val()
        print(f"[*] Deduced Hidden Altar Value: {altar_val}")
        self.pgrid()

        ops = self.simulate_sweep(root, altar_val)
        print(f"\n[*] Executing {len(ops)} sequence... (budget:{self.bud})")
        
        self.execute(ops)

        # Verify
        print(f"\n[*] Extinguished 40/40. Going to altar... (budget:{self.bud})")
        self.goto(4, 7)
        self.pgrid()

        # Final Cleanup Check - if any cell failed due to dropped packet
        non0 = [(r,c) for r in range(ROWS) for c in range(COLS) if self.g.get((r,c)) in ('*', '~')]
        if non0:
            print(f"[!] DIVERGENCE DETECTED! Failed to zero cells: {non0}")
            for r, c in non0:
                print(f"[!] Cell {(r,c)} is {self.g.get((r,c))}.")
            
            print("[*] Switching to interactive Fallback to avoid wasting connection...")
            try:
                while True:
                    cmd_in = input("> ").strip()
                    if cmd_in in ('q','quit'): break
                    if cmd_in == 'g': self.pgrid(); continue
                    if cmd_in: print(self.cmd(cmd_in))
            except: pass
            self.r.close()
            return

        # Pray
        print(f"\n[*] Praying... (budget:{self.bud})")
        self.r.sendline(b'pray')
        try:
            resp = self.r.recvall(timeout=5).decode()
        except:
            resp = "Server closed connection"
            
        print("=== FINAL SERVER RESPONSE ===")
        print(resp)
        print("=============================")
        
        for line in resp.split('\n'):
            ll = line.lower()
            if any(x in ll for x in ['flag','hto{','ctf{','congratul','well','solve']):
                print(f"\n{'='*60}\n[FLAG] {line.strip()}\n{'='*60}")
        try:
            while True:
                c = input("> ").strip()
                if c in ('q','quit'): break
                if c == 'g': self.pgrid(); continue
                if c: print(self.cmd(c))
        except: pass
        self.r.close()

if __name__ == '__main__':
    Solver().solve()
```
