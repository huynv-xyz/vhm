# Git First Principles
## Hiểu Git từ bản chất thay vì học thuộc command

> Mục tiêu của tài liệu này: sau khi đọc xong, bạn không cần nhớ hàng chục lệnh Git.  
> Khi gặp một command lạ, bạn có thể tự suy luận bằng 3 câu hỏi:
>
> 1. Command này **tạo object mới** hay không?
> 2. Command này **di chuyển pointer/ref nào**?
> 3. Command này thay đổi **HEAD, Index hay Working Tree**?

---

# Mục lục

1. First Principles Thinking là gì?
2. Bài toán gốc mà Git giải quyết
3. Mô hình tối thiểu của Git
4. Content-addressable storage
5. Blob
6. Tree
7. Snapshot
8. Commit
9. Commit Graph và DAG
10. Branch thực chất là gì?
11. HEAD là gì?
12. Detached HEAD
13. Working Tree
14. Index / Staging Area
15. Bản chất của `git add`
16. Bản chất của `git commit`
17. `git diff` đang so sánh cái gì?
18. `git status` suy ra như thế nào?
19. Branching
20. Fast-forward
21. Merge
22. Conflict
23. Rebase
24. Cherry-pick
25. Amend
26. Reset
27. Restore
28. Revert
29. Reflog
30. Remote
31. Fetch
32. Pull
33. Push
34. Force push và `--force-with-lease`
35. Tag
36. `.git` directory
37. Git object immutable có ý nghĩa gì?
38. Git GC và object “mất”
39. Những command Git quy về primitive nào?
40. Mental model tổng hợp
41. Workflow thực tế
42. Các case thường gặp
43. Bài lab để hiểu Git sâu
44. Checklist tự kiểm tra
45. Cheat sheet cuối cùng

---

# 1. First Principles Thinking là gì?

Thông thường người ta học Git kiểu:

```bash
git add .
git commit -m "..."
git push
git pull
git checkout
git rebase
```

Vấn đề là:

- biết lệnh nhưng không biết Git đang làm gì;
- gặp conflict thì hoảng;
- reset nhầm thì sợ;
- rebase bị conflict thì không biết lịch sử đang biến đổi thế nào;
- thấy detached HEAD tưởng repo hỏng;
- force push không hiểu tại sao nguy hiểm.

First Principles Thinking làm ngược lại.

Ta hỏi:

> Nếu phải tự thiết kế một Version Control System từ đầu, hệ thống tối thiểu cần những primitive nào?

Sau đó từ các primitive đó, ta tự suy ra các tính năng cao hơn.

Git nhìn từ first principles chỉ xoay quanh vài thứ:

```text
DATA
├── blob
├── tree
└── commit

GRAPH
└── parent relationship

POINTER
├── branch/ref
└── HEAD

WORKING STATE
├── HEAD snapshot
├── Index
└── Working Tree

DISTRIBUTION
├── local refs
├── remote-tracking refs
└── object transfer
```

Nếu hiểu các khối này, hầu hết Git trở nên logic.

---

# 2. Bài toán gốc mà Git giải quyết

Giả sử project ban đầu:

```text
project/
├── A.java
└── B.java
```

Ngày 1:

```text
A.java = version 1
B.java = version 1
```

Ngày 2:

```text
A.java = version 2
B.java = version 1
```

Ngày 3:

```text
A.java = version 3
B.java = version 2
```

Cách ngây thơ nhất để lưu version:

```text
backup-01/
backup-02/
backup-final/
backup-final-2/
backup-final-real/
```

Nhưng ngay lập tức có các câu hỏi:

- Version nào tạo trước?
- Version nào dựa trên version nào?
- Ai thay đổi?
- Thay đổi gì?
- Có thể quay lại không?
- Hai người sửa song song thì sao?
- Làm sao hợp nhất?
- Làm sao chia sẻ lịch sử cho máy khác?
- Làm sao kiểm tra dữ liệu có bị thay đổi không?

Từ đây ta suy ra một Version Control System cần giải quyết ít nhất:

```text
1. Lưu content
2. Lưu structure
3. Lưu snapshot
4. Lưu history
5. Có identity cho mỗi version
6. Cho phép phát triển song song
7. Cho phép hợp nhất lịch sử
8. Cho phép đồng bộ giữa nhiều máy
```

Git chính là một cách rất đẹp để giải các bài toán đó.

---

# 3. Mô hình tối thiểu của Git

Nếu nén Git xuống mức tối thiểu:

```text
blob   = nội dung file
tree   = cấu trúc thư mục
commit = snapshot + parent + metadata
ref    = tên trỏ tới commit
HEAD   = vị trí hiện tại
index  = snapshot chuẩn bị commit
working tree = file đang edit
```

Mental model tổng quát:

```text
                     Git Repository
                           │
              ┌────────────┴────────────┐
              │                         │
          OBJECTS                     REFS
              │                         │
      blob / tree / commit        main / feature / tag
              │                         │
              └──────── Commit DAG ─────┘


                  Developer Working State

                HEAD snapshot
                     │
                     ▼
                   Index
                     │
                     ▼
                Working Tree
```

---

# 4. Content-addressable storage

Một trong những ý tưởng quan trọng nhất của Git:

> Git định danh dữ liệu dựa trên chính nội dung của dữ liệu.

Conceptually:

```text
object_id = HASH(type + content)
```

Ví dụ file chứa:

```text
Hello Git
```

Git tạo ra một hash đại diện cho content đó.

Tư duy:

```text
content
   ↓
 hash
   ↓
object id
```

Nếu content giống nhau:

```text
same content
    ↓
same object identity
```

Điều này tạo ra nhiều tính chất hay:

### Integrity

Nếu dữ liệu bị thay đổi, hash thay đổi.

### Deduplication

Nội dung giống nhau có thể reuse object.

### Immutable object

Một object có identity phụ thuộc vào content.

Muốn thay content thì về bản chất ta tạo object khác.

Đây là nền tảng giúp hiểu:

- amend;
- rebase;
- cherry-pick;
- reset;
- reflog;
- garbage collection.

---

# 5. Blob

Blob là primitive đầu tiên.

Blob lưu:

```text
file content
```

Blob không quan tâm:

- tên file;
- path;
- extension;
- file nằm folder nào.

Ví dụ:

```text
hello.txt
```

chứa:

```text
hello
```

Git nhìn gần giống:

```text
blob ABC123
content = "hello"
```

Nếu có:

```text
a.txt = hello
b.txt = hello
```

về mặt content Git có thể dùng cùng một blob.

Điểm cần nhớ:

> Blob biết content, nhưng không biết filename.

---

# 6. Tree

Nếu blob không biết filename thì cần object khác để mô tả structure.

Đó là `tree`.

Ví dụ:

```text
project/
├── A.java
└── B.java
```

có thể được mô hình hóa:

```text
tree ROOT
├── A.java → blob AAA
└── B.java → blob BBB
```

Nếu có nested directory:

```text
project/
└── src/
    ├── A.java
    └── B.java
```

thì:

```text
tree ROOT
└── src → tree SRC
          ├── A.java → blob AAA
          └── B.java → blob BBB
```

Vậy:

```text
blob = content
tree = mapping tên/path → object
```

Tree cho Git khả năng mô tả toàn bộ filesystem của project.

---

# 7. Snapshot

Một root tree đại diện cho:

> Toàn bộ trạng thái của repository tại một thời điểm.

Đó chính là snapshot.

Nhiều người nghĩ Git lưu lịch sử dưới dạng:

```text
commit 1 = diff
commit 2 = diff
commit 3 = diff
```

Mental model tốt hơn:

```text
commit 1 → snapshot S1
commit 2 → snapshot S2
commit 3 → snapshot S3
```

Ví dụ:

Commit 1:

```text
A.java → blob AAA
B.java → blob BBB
```

Sửa A.java.

Commit 2:

```text
A.java → blob CCC
B.java → blob BBB
```

Blob BBB được reuse.

Do đó:

> Git có mental model snapshot, nhưng storage vẫn hiệu quả vì các object không đổi được reuse.

---

# 8. Commit

Snapshot chưa đủ để tạo history.

Giả sử có:

```text
snapshot A
snapshot B
snapshot C
```

Ta chưa biết thứ tự.

Do đó cần `commit`.

Một commit về bản chất chứa:

```text
commit
├── tree
├── parent
├── author
├── committer
├── timestamp
└── message
```

Ví dụ:

```text
Commit C
├── tree → snapshot hiện tại
├── parent → Commit B
├── author → Huy
└── message → Add payment API
```

Điểm cực quan trọng:

> Commit không phải là diff.

Commit chứa:

```text
snapshot
+
parent pointer
+
metadata
```

Diff được Git tính bằng cách so hai snapshot:

```text
diff(parent, commit)
```

---

# 9. Commit Graph và DAG

Nếu:

```text
B.parent = A
C.parent = B
```

ta có:

```text
A ← B ← C
```

Arrow trỏ về parent, tức là quá khứ.

Nếu branch:

```text
        D ← E
       /
A ← B ← C
```

ta có graph.

Git history không phải linked list đơn giản.

Nó là:

```text
Directed Acyclic Graph
```

viết tắt:

```text
DAG
```

### Directed

Commit trỏ tới parent.

### Acyclic

History không tạo vòng tròn.

Không thể:

```text
A → B → C → A
```

vì identity commit phụ thuộc parent và graph đi về quá khứ.

Đây là nền tảng của:

- branch;
- merge;
- rebase;
- ancestor;
- merge-base;
- revision traversal.

---

# 10. Branch thực chất là gì?

Một trong những hiểu lầm phổ biến nhất:

> Branch là một bản copy của code.

Sai.

Branch chỉ là:

> Một cái tên trỏ tới một commit.

Ví dụ:

```text
A ← B ← C
        ↑
       main
```

`main` chỉ lưu identity của C.

Nếu tạo:

```bash
git branch feature
```

ta có:

```text
A ← B ← C
        ↑
       main
        ↑
     feature
```

Conceptually:

```text
refs/heads/main    = hash(C)
refs/heads/feature = hash(C)
```

Không copy file.

Không copy repository.

Không copy history.

Đó là lý do Git branch rất nhẹ.

---

# 11. HEAD là gì?

Giả sử:

```text
main    → C
feature → C
```

Git cần biết developer đang làm việc trên branch nào.

Đó là `HEAD`.

Thông thường:

```text
HEAD → main → C
```

HEAD thường không trỏ trực tiếp tới commit mà trỏ tới branch hiện tại.

Ví dụ:

```text
HEAD
 ↓
main
 ↓
 C
```

Khi commit mới:

```text
A ← B ← C ← D
            ↑
           main
            ↑
           HEAD
```

Conceptually:

```text
D.parent = C
main = D
```

HEAD vẫn trỏ tới main.

Branch main di chuyển.

---

# 12. Detached HEAD

HEAD cũng có thể trỏ trực tiếp tới commit:

```text
HEAD → C
```

thay vì:

```text
HEAD → main → C
```

Đó là detached HEAD.

Ví dụ:

```bash
git switch --detach <commit>
```

Mental model:

```text
main → C

HEAD → B
```

Nếu commit khi detached:

```text
A ← B ← D
    ↑    ↑
   ...  HEAD
```

D tồn tại nhưng không có branch bình thường trỏ vào nó.

Nếu sau đó switch sang main, D có thể trở thành unreachable từ normal branch refs.

Repo không hỏng.

Đơn giản là:

> HEAD không attach vào một named branch.

---

# 13. Working Tree

Git object database không phải nơi developer edit trực tiếp.

Developer cần file thật:

```text
src/
pom.xml
README.md
```

Đó là Working Tree.

Working Tree là representation có thể edit của project.

Ví dụ:

```text
HEAD snapshot:
A.java = v1

Working Tree:
A.java = v2
```

Git thấy:

```text
Working Tree != HEAD
```

nên biết file modified.

---

# 14. Index / Staging Area

Nếu chỉ có:

```text
HEAD
Working Tree
```

thì Git gặp bài toán:

Bạn sửa 10 file nhưng chỉ muốn commit 3 file.

Git cần một state ở giữa.

Đó là:

```text
Index
```

hay:

```text
Staging Area
```

Mental model quan trọng nhất khi làm Git hằng ngày:

```text
HEAD
 ↓
Index
 ↓
Working Tree
```

Ba trạng thái:

```text
HEAD snapshot    = commit hiện tại
Index            = snapshot sắp commit
Working Tree     = trạng thái bạn đang edit
```

---

# 15. Bản chất của `git add`

Người mới thường nghĩ:

```text
git add = đánh dấu file
```

Mental model sâu hơn:

> `git add` lấy content hiện tại trong Working Tree và cập nhật Index.

Ví dụ:

Ban đầu:

```text
HEAD    = A:v1
Index   = A:v1
Working = A:v1
```

Sửa file:

```text
HEAD    = A:v1
Index   = A:v1
Working = A:v2
```

Chạy:

```bash
git add A
```

thành:

```text
HEAD    = A:v1
Index   = A:v2
Working = A:v2
```

Sau đó sửa tiếp:

```text
HEAD    = A:v1
Index   = A:v2
Working = A:v3
```

Nếu commit lúc này:

```text
commit chứa A:v2
```

Không phải A:v3.

Đây là bằng chứng Index thật sự là snapshot độc lập.

---

# 16. Bản chất của `git commit`

`git commit` không lấy toàn bộ Working Tree.

Nó lấy:

```text
Index
```

Conceptually:

```text
Index
  ↓
create tree
  ↓
create commit
  ↓
commit.parent = current HEAD commit
  ↓
move current branch ref
```

Ví dụ:

Trước:

```text
HEAD → main → C

Index = snapshot S
```

Commit:

```text
D.tree = S
D.parent = C
```

sau đó:

```text
HEAD → main → D
```

Tóm gọn:

> `git add` xây snapshot.  
> `git commit` freeze snapshot đó thành lịch sử.

---

# 17. `git diff` đang so sánh cái gì?

Vì có 3 state:

```text
HEAD
Index
Working Tree
```

nên diff chỉ là compare giữa các state.

## `git diff`

```text
Working Tree
vs
Index
```

Nó trả lời:

> Tôi đã sửa gì nhưng chưa stage?

## `git diff --staged`

```text
Index
vs
HEAD
```

Nó trả lời:

> Commit tiếp theo sẽ khác commit hiện tại như thế nào?

## `git diff HEAD`

```text
Working Tree
vs
HEAD
```

Nó trả lời:

> Tổng thay đổi hiện tại so với commit đang đứng là gì?

Mental model:

```text
HEAD -------- Index -------- Working
   staged diff     unstaged diff
```

---

# 18. `git status` suy ra như thế nào?

`git status` gần như chỉ compare:

```text
HEAD ↔ Index
Index ↔ Working Tree
```

Nếu:

```text
HEAD != Index
```

thì có staged changes.

Nếu:

```text
Index != Working Tree
```

thì có unstaged changes.

Nếu file ở Working Tree nhưng không có trong Index/HEAD:

```text
untracked
```

Không có ma thuật.

---

# 19. Branching

Ban đầu:

```text
A---B---C main
```

Tạo feature:

```text
A---B---C
        ↑
       main
        ↑
     feature
```

Switch feature:

```text
HEAD → feature → C
```

Commit D, E:

```text
A---B---C main
         \
          D---E feature
```

Về bản chất:

```text
D.parent = C
E.parent = D
feature = E
```

main không đổi.

Branching chỉ là:

```text
tạo ref
+
move HEAD sang ref
+
các commit sau làm ref đó tiến lên
```

---

# 20. Fast-forward

Giả sử:

```text
A---B---C main
         \
          D---E feature
```

main không có commit mới.

Muốn merge feature vào main.

Không cần tạo merge commit.

Chỉ cần:

```text
main → E
```

Kết quả:

```text
A---B---C---D---E
                ↑
               main
```

Đó là fast-forward.

Bản chất:

> Chỉ di chuyển branch pointer về một descendant.

---

# 21. Merge

Giả sử hai branch cùng phát triển:

```text
      D---E feature
     /
A---B---C main
```

Muốn kết hợp hai line history.

Git tìm common ancestor:

```text
B
```

gọi là:

```text
merge base
```

Sau đó so:

```text
B → C
B → E
```

Nếu hợp nhất được, tạo snapshot mới M.

Merge commit:

```text
      D---E
     /     \
A---B---C---M
```

M khác normal commit ở điểm:

```text
M.parents = [C, E]
```

Một merge commit đơn giản là:

> Commit có nhiều parent.

---

# 22. Conflict

Giả sử common base:

```java
int x = 1;
```

main:

```java
int x = 2;
```

feature:

```java
int x = 3;
```

Git có thông tin:

```text
base   = 1
ours   = 2
theirs = 3
```

Nhưng Git không biết business intent.

Kết quả đúng có thể là:

```text
2
3
5
10
hoặc xóa dòng
```

Do đó Git dừng.

Conflict có nghĩa:

> Dữ liệu hiện có không đủ để Git tự quyết định kết quả.

Conflict không phải error của Git.

Nó là ambiguity.

---

# 23. Rebase

Giả sử:

```text
A---B---C main
     \
      D---E feature
```

Ta muốn feature giống như bắt đầu từ C:

```text
A---B---C---D'---E'
```

Tại sao không sửa D.parent từ B thành C?

Vì commit identity phụ thuộc content:

```text
commit hash
depends on
tree + parent + metadata
```

Đổi parent thì hash đổi.

Commit cũ không được mutate.

Do đó Git phải tạo commit mới.

Conceptually:

```text
change D = diff(B, D)
apply change D onto C
→ create D'

change E = diff(D, E)
apply change E onto D'
→ create E'
```

Vậy:

```text
D != D'
E != E'
```

dù source code có thể giống.

Bản chất rebase:

> Replay changes lên một base mới và tạo commit identity mới.

---

# 24. Cherry-pick

Giả sử:

```text
A---B---C main

X---Y---Z feature
```

Muốn lấy thay đổi của Y vào main.

Git không move Y.

Nó tính:

```text
diff(Y.parent, Y)
```

rồi apply lên C.

Kết quả:

```text
A---B---C---Y'
```

Y' là commit mới.

Bản chất:

> Cherry-pick copy effect/change, không copy identity của commit.

---

# 25. Amend

Giả sử:

```text
A---B---C
        ↑
       main
```

Chạy:

```bash
git commit --amend
```

Git không sửa C.

Nó tạo C':

```text
A---B---C'
```

sau đó:

```text
main → C'
```

C cũ vẫn có thể tồn tại trong object database.

Bản chất:

> Amend = tạo replacement commit + move ref.

---

# 26. Reset

Reset dễ hiểu nếu nhớ 3 layer:

```text
HEAD/Branch
Index
Working Tree
```

Giả sử target là commit X.

## `git reset --soft X`

Thay đổi:

```text
branch → X
```

Giữ:

```text
Index
Working Tree
```

Mental model:

```text
Branch      changed
Index       unchanged
Working     unchanged
```

## `git reset --mixed X`

Thay đổi:

```text
branch → X
Index = snapshot(X)
```

Giữ Working Tree.

```text
Branch      changed
Index       changed
Working     unchanged
```

`--mixed` là default.

## `git reset --hard X`

Thay cả 3:

```text
branch → X
Index = X
Working Tree = X
```

```text
Branch      changed
Index       changed
Working     changed
```

Bảng nhớ:

```text
                  Branch   Index   Working
--soft              ✓
--mixed             ✓       ✓
--hard              ✓       ✓       ✓
```

Không cần học thuộc nếu hiểu layer.

---

# 27. Restore

`git restore` chủ yếu thao tác file-level state.

Ví dụ:

```bash
git restore file.txt
```

Conceptually:

```text
Working Tree file
←
source state
```

Mặc định thường lấy từ Index.

Còn:

```bash
git restore --staged file.txt
```

thay Index cho file đó.

Mental model:

```text
restore = copy file state từ một layer/source sang layer khác
```

Nó khác `reset` ở chỗ reset thường liên quan ref/commit position rộng hơn.

---

# 28. Revert

Giả sử:

```text
A---B---C
```

Muốn undo effect của C nhưng không rewrite history.

Git tạo D:

```text
A---B---C---D
```

D chứa inverse change của C.

Bản chất:

```text
revert = create new history
```

Trong khi:

```text
reset = move pointer
```

Do đó shared branch thường dùng revert vì lịch sử cũ vẫn giữ nguyên.

---

# 29. Reflog

Giả sử:

```text
main → C
```

sau đó:

```bash
git reset --hard A
```

thành:

```text
main → A
```

C có thể vẫn tồn tại trong object database.

Vấn đề chỉ là branch không còn trỏ tới C.

Git ghi lại lịch sử thay đổi refs/HEAD trong reflog.

Ví dụ:

```text
main:
C → A
```

Do đó:

```bash
git reflog
```

có thể cho biết commit cũ.

Sau đó có thể recover:

```bash
git reset --hard <old-hash>
```

First principle:

> Nhiều tình huống “mất commit” thực ra chỉ là mất pointer tới commit.

Reflog giúp tìm pointer history.

---

# 30. Remote

Git là distributed version control.

Repository local đã chứa history.

Remote không phải “server điều khiển local Git”.

Remote chỉ là repository khác mà local biết cách nói chuyện.

Ví dụ remote GitHub có:

```text
main → C
```

Local có:

```text
main → B
```

Local còn giữ knowledge về remote:

```text
origin/main → C
```

Điểm quan trọng:

> `origin/main` là remote-tracking ref trong local repository.

Nó không phải branch live trực tiếp trên server.

---

# 31. Fetch

`git fetch` về bản chất:

```text
1. download objects mình chưa có
2. update remote-tracking refs
```

Ví dụ server:

```text
main → F
```

Local trước fetch:

```text
origin/main → C
```

Sau fetch:

```text
origin/main → F
```

Nhưng local branch:

```text
main → B
```

có thể vẫn đứng yên.

Fetch không merge code vào branch hiện tại.

Đây là một trong những command an toàn nhất để quan sát remote.

---

# 32. Pull

Mental model đơn giản:

```text
git pull
≈
git fetch
+
integrate
```

Integrate thường là:

```text
merge
```

hoặc nếu config:

```text
rebase
```

Do đó nên nghĩ:

```text
pull = fetch + merge/rebase
```

Thay vì coi pull là primitive riêng.

---

# 33. Push

Giả sử local:

```text
A---B---C---D---E main
```

remote:

```text
A---B---C main
```

Push muốn yêu cầu remote:

```text
main C → E
```

Nếu C là ancestor của E, remote có thể fast-forward.

Bản chất push:

```text
transfer objects
+
request remote ref update
```

---

# 34. Force push và `--force-with-lease`

Giả sử remote có:

```text
A---B---C---F
```

Local sau rebase:

```text
A---B---C---D'---E'
```

Push bình thường bị reject vì remote ref sẽ không fast-forward.

## `--force`

Conceptually:

```text
remote:
hãy đặt branch vào commit tôi chỉ định,
không quan tâm state hiện tại
```

Nguy hiểm vì có thể overwrite work người khác.

## `--force-with-lease`

Mental model giống optimistic locking.

Conceptually:

```sql
UPDATE branch
SET commit = my_new_commit
WHERE commit = expected_old_commit;
```

Tức:

> Chỉ force nếu remote vẫn đang ở state mà local tin rằng nó đang ở.

Nếu người khác vừa push commit mới, condition fail.

Do đó `--force-with-lease` an toàn hơn nhiều so với `--force`.

---

# 35. Tag

Branch:

```text
pointer di chuyển theo commit mới
```

Tag thường:

```text
pointer cố định tới một commit/object
```

Ví dụ:

```text
v1.0.0 → C
```

Sau khi main đi tiếp:

```text
A---B---C---D---E
        ↑       ↑
      v1.0.0   main
```

Tag dùng để đánh dấu một point lịch sử quan trọng.

---

# 36. `.git` directory

Một Git repository thực chất nằm trong:

```text
.git/
```

Các phần quan trọng:

```text
.git/
├── HEAD
├── index
├── objects/
└── refs/
    ├── heads/
    └── tags/
```

Mental model:

```text
objects/ = immutable-ish data
refs/    = mutable pointers
HEAD     = vị trí hiện tại
index    = staging snapshot
```

Nếu hiểu bốn phần này, Git giảm rất nhiều “ma thuật”.

---

# 37. Git object immutable có ý nghĩa gì?

Git object identity dựa trên content.

Nếu object thay đổi:

```text
identity thay đổi
```

Do đó Git thường không sửa object cũ.

Nó:

```text
create new object
+
move pointer
```

Từ nguyên lý này tự suy ra:

```text
amend
→ tạo commit mới

rebase
→ tạo chuỗi commit mới

cherry-pick
→ tạo commit mới

merge
→ tạo commit mới

reset
→ thường không sửa commit
→ chỉ di chuyển ref/state
```

Đây là một trong những nguyên lý quan trọng nhất của Git.

---

# 38. Git GC và object “mất”

Sau reset/rebase/amend, commit cũ có thể không còn reachable từ branch.

Ví dụ:

Trước:

```text
A---B---C
        ↑
       main
```

Sau amend:

```text
A---B---C'
        ↑
       main
```

C cũ có thể vẫn nằm trong object database.

Nếu còn reflog reference, vẫn có thể recover.

Sau một thời gian, unreachable objects có thể được garbage collection.

Do đó:

> Git rất recoverable, nhưng không phải vô hạn.

Nếu vừa thao tác nhầm:

```text
đừng hoảng
đừng tạo thêm quá nhiều thay đổi
kiểm tra reflog trước
```

---

# 39. Những command Git quy về primitive nào?

## `git branch`

```text
create/move ref
```

## `git switch`

```text
change HEAD target
+
update Working Tree/Index phù hợp
```

## `git add`

```text
Working Tree → Index
```

## `git commit`

```text
Index → tree → commit
+
move branch ref
```

## `git merge`

```text
combine graph histories
+
possibly create multi-parent commit
```

## `git rebase`

```text
replay changes
+
create replacement commits
+
move ref
```

## `git cherry-pick`

```text
apply one commit's effect
+
create new commit
```

## `git amend`

```text
create replacement commit
+
move ref
```

## `git reset`

```text
move ref
+
optionally replace Index
+
optionally replace Working Tree
```

## `git restore`

```text
copy file state giữa layers
```

## `git revert`

```text
create inverse commit
```

## `git fetch`

```text
download objects
+
update remote refs
```

## `git push`

```text
upload objects
+
request remote ref movement
```

---

# 40. Mental model tổng hợp

Nếu chỉ giữ một hình trong đầu, hãy giữ hình này:

```text
                         GIT


                   OBJECT DATABASE
              ┌─────────────────────┐
              │ blob                │
              │ tree                │
              │ commit              │
              └──────────┬──────────┘
                         │
                         ▼
                    COMMIT DAG

                  A ← B ← C ← D
                          ↑
                         main
                          ↑
                         HEAD


                 WORKING STATE

           HEAD snapshot
                │
                │ compare
                ▼
              Index
                │
                │ compare
                ▼
           Working Tree
```

Và nhớ:

```text
Objects = dữ liệu
Refs    = pointer
HEAD    = mình đang ở đâu
Index   = commit tiếp theo trông như thế nào
Working = mình đang sửa gì
```

---

# 41. Workflow thực tế

Một workflow bình thường:

```bash
git fetch
git switch main
git pull --ff-only

git switch -c feature/payment

# edit code

git status
git diff

git add src/payment
git diff --staged

git commit -m "Add payment validation"

git fetch
git rebase origin/main

git push -u origin feature/payment
```

Đừng chỉ chạy lệnh.

Hãy nhìn mental state:

### Sau `switch -c`

```text
HEAD → feature → commit hiện tại
```

### Sau sửa code

```text
Working != Index
```

### Sau `git add`

```text
Index != HEAD
```

### Sau commit

```text
new commit created
feature ref moved
```

### Sau rebase

```text
feature commits replayed trên origin/main
new commit identities created
```

### Sau push

```text
remote nhận objects
remote branch ref được update
```

---

# 42. Các case thường gặp

## Case 1: Commit nhầm file

Bạn đã stage:

```text
A
B
C
```

nhưng không muốn B.

Dùng:

```bash
git restore --staged B
```

Mental model:

```text
Index(B) ← HEAD(B)
```

Working file B vẫn giữ.

---

## Case 2: Commit rồi mới phát hiện thiếu file

Nếu commit chưa public/shared:

```bash
git add missing-file
git commit --amend
```

Mental model:

```text
create C'
move branch C → C'
```

Không phải sửa C.

---

## Case 3: Muốn bỏ commit gần nhất nhưng giữ code

```bash
git reset --soft HEAD~1
```

Mental model:

```text
move branch backward
keep Index
keep Working Tree
```

---

## Case 4: Muốn bỏ commit và unstage nhưng giữ code

```bash
git reset HEAD~1
```

tức mixed.

Mental model:

```text
move branch backward
Index = target
Working Tree giữ code
```

---

## Case 5: Muốn xóa sạch commit và code

```bash
git reset --hard HEAD~1
```

Mental model:

```text
branch
Index
Working Tree
```

đều quay lại target.

Nguy hiểm vì Working Tree bị overwrite.

---

## Case 6: Đã push lên shared branch và muốn undo

Ưu tiên:

```bash
git revert <commit>
```

Thay vì reset + force.

Vì revert giữ lịch sử public.

---

## Case 7: Rebase xong commit hash thay đổi

Đúng.

Vì:

```text
parent thay đổi
→ commit content identity thay đổi
→ hash thay đổi
```

Đó là bản chất rebase.

---

## Case 8: Reset nhầm mất commit

Đầu tiên:

```bash
git reflog
```

Tìm commit cũ.

Sau đó tạo branch cứu:

```bash
git branch rescue <old-hash>
```

Hoặc reset trở lại.

---

## Case 9: MR đã mở, muốn bỏ một file khỏi MR

MR về bản chất compare branch source với target.

Nếu file đã commit trong feature branch, hãy đưa file đó về state giống target branch rồi commit.

Ví dụ:

```bash
git restore --source=origin/main -- path/to/file
git add path/to/file
git commit -m "Restore file from main"
```

Khi diff giữa feature và main không còn khác file đó, MR sẽ không còn hiển thị thay đổi file.

---

## Case 10: Conflict khi rebase

Rebase đang replay từng commit.

Mental model:

```text
take old commit effect
apply lên new base
```

Nếu effect không apply được:

```text
conflict
```

Flow:

```bash
# resolve file
git add <resolved-files>
git rebase --continue
```

Nếu không muốn nữa:

```bash
git rebase --abort
```

---

# 43. Bài lab để hiểu Git sâu

Đây là phần nên làm thật.

---

## Lab 1: Nhìn blob

```bash
mkdir git-lab
cd git-lab
git init

echo "hello" > a.txt

git hash-object a.txt
```

Quan sát hash.

Sau đó:

```bash
echo "hello2" > a.txt
git hash-object a.txt
```

Hash thay đổi.

Kết luận:

```text
content thay đổi
→ identity thay đổi
```

---

## Lab 2: Tự ghi object

```bash
echo "hello" | git hash-object -w --stdin
```

Sau đó:

```bash
git cat-file -t <hash>
git cat-file -p <hash>
```

Bạn đang trực tiếp thao tác Git object database.

---

## Lab 3: Nhìn commit object

```bash
git add a.txt
git commit -m "first"

git cat-file -p HEAD
```

Quan sát:

```text
tree ...
author ...
committer ...
```

Nếu có parent sẽ thấy:

```text
parent ...
```

Kết luận:

> Commit chỉ là structured object.

---

## Lab 4: Nhìn tree

```bash
git cat-file -p HEAD^{tree}
```

Hoặc:

```bash
git ls-tree HEAD
```

Quan sát tree trỏ tới blob.

---

## Lab 5: Nhìn branch ref

```bash
cat .git/HEAD
```

Có thể thấy:

```text
ref: refs/heads/main
```

Sau đó:

```bash
cat .git/refs/heads/main
```

sẽ thấy commit hash.

Kết luận:

```text
HEAD → branch ref → commit
```

---

## Lab 6: Chứng minh branch chỉ là pointer

```bash
git branch feature
```

Xem:

```bash
cat .git/refs/heads/main
cat .git/refs/heads/feature
```

Hai ref có thể cùng hash.

Không có source code nào bị copy.

---

## Lab 7: Chứng minh Index độc lập

```bash
echo "v1" > a.txt
git add a.txt
git commit -m "v1"

echo "v2" > a.txt
git add a.txt

echo "v3" > a.txt
```

Bây giờ:

```bash
git diff
git diff --staged
```

Bạn sẽ thấy:

```text
HEAD = v1
Index = v2
Working = v3
```

Sau đó:

```bash
git commit -m "v2"
```

Commit chứa v2.

---

## Lab 8: Rebase tạo commit mới

Tạo graph:

```text
main:    A---B---C
              \
feature:       D---E
```

Lưu hash D/E.

Sau:

```bash
git rebase main
```

Xem hash mới.

Bạn sẽ thấy D/E trở thành D'/E'.

---

## Lab 9: Reset và reflog

Tạo vài commit:

```text
A---B---C
```

Chạy:

```bash
git reset --hard A
```

Sau đó:

```bash
git log
```

B/C biến mất khỏi branch.

Nhưng:

```bash
git reflog
```

vẫn thấy lịch sử HEAD.

Recover:

```bash
git branch rescue <hash-C>
```

---

## Lab 10: Fast-forward vs merge commit

Tạo branch feature từ main.

Trường hợp 1:

main không đi tiếp.

Merge feature:

```text
fast-forward
```

Trường hợp 2:

main và feature cùng có commit mới.

Merge:

```text
merge commit có 2 parents
```

Sau đó:

```bash
git cat-file -p <merge-commit>
```

xem hai dòng `parent`.

---

# 44. Checklist tự kiểm tra

Nếu trả lời được các câu này mà không tra tài liệu, bạn đã hiểu Git khá sâu.

### Object model

- Blob chứa gì?
- Blob có biết filename không?
- Tree dùng để làm gì?
- Commit chứa diff hay snapshot?
- Commit có biết child không?
- Vì sao thay parent lại thay commit hash?

### Graph

- Vì sao Git history là DAG?
- Branch là gì?
- Branch có copy source code không?
- Merge commit khác commit thường ở đâu?
- Merge base là gì?

### Working state

- HEAD là gì?
- Detached HEAD nghĩa là gì?
- Index là gì?
- `git add` thật sự làm gì?
- `git commit` lấy source từ Working Tree hay Index?

### Diff

- `git diff` so gì với gì?
- `git diff --staged` so gì với gì?

### History rewriting

- Vì sao amend tạo hash mới?
- Vì sao rebase tạo commit mới?
- Cherry-pick copy commit hay copy effect?
- Reset có sửa commit object không?

### Recovery

- Reflog lưu gì?
- Vì sao reset nhầm thường vẫn cứu được?

### Remote

- `origin/main` nằm local hay remote?
- Fetch khác pull thế nào?
- Push về bản chất làm gì?
- Fast-forward là gì?
- Vì sao push non-fast-forward bị reject?
- `--force-with-lease` giống optimistic locking thế nào?

---

# 45. Cheat sheet cuối cùng

## Bảy khái niệm cần nhớ

```text
1. Blob
   = file content

2. Tree
   = directory structure

3. Commit
   = snapshot + parent + metadata

4. DAG
   = history graph

5. Ref / Branch
   = named pointer to commit

6. HEAD
   = current position

7. Index + Working Tree
   = prepared snapshot + editable files
```

## Ba layer làm việc

```text
HEAD
 ↓
Index
 ↓
Working Tree
```

## Hai loại operation lớn

Hầu hết Git command chỉ làm một hoặc cả hai:

```text
1. Create new objects
2. Move refs/state
```

## Cách suy luận mọi command

Khi gặp một command Git, hãy hỏi:

```text
1. Nó đọc state từ đâu?

   HEAD?
   Index?
   Working Tree?
   commit khác?
   remote?

2. Nó tạo object mới không?

   blob?
   tree?
   commit?

3. Nó move pointer nào?

   HEAD?
   branch?
   tag?
   remote-tracking ref?

4. Nó rewrite Working Tree hoặc Index không?

5. Commit identity cũ còn tồn tại hay bị thay bằng commit mới?
```

---

# Kết luận

Git không thực sự là một tập hợp command khó nhớ.

Bản chất Git có thể nén xuống:

```text
content-addressable object database
+
commit DAG
+
mutable refs
+
three working states
+
object exchange giữa repositories
```

Mental model cuối cùng:

```text
                 immutable-ish DATA
               blob / tree / commit
                        │
                        ▼
                    Commit DAG
                        │
                        ▼
                  refs / branches
                        │
                        ▼
                       HEAD


HEAD snapshot
     │
     ▼
   Index
     │
     ▼
Working Tree


Remote operations:
fetch = get objects + update remote refs
push  = send objects + request ref update
```

Nếu chỉ nhớ một nguyên lý, hãy nhớ:

> Git chủ yếu không sửa lịch sử cũ.  
> Git tạo object mới rồi di chuyển pointer.

Và khi gặp bất kỳ command nào, đừng hỏi:

> “Cú pháp command này là gì?”

Hãy hỏi:

> “Nó đang tạo object gì, di chuyển pointer nào, và thay đổi HEAD / Index / Working Tree ra sao?”

Khi tư duy được như vậy, Git không còn là một bộ command phải học thuộc. Nó trở thành một hệ thống có thể tự suy luận.
