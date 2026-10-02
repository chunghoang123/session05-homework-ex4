# Bài 4: Mô phỏng quy trình Hotfix & Gitflow thực tế

## 1. Mục tiêu
- Áp dụng mô hình Gitflow: `main` (stable) + `develop` (tương lai) + `hotfix/*` (vá nóng).
- Tạo, kiểm thử và tích hợp nhánh `hotfix/v1.0.1` trực tiếp từ `main`.
- Gắn tag `v1.0.1`, merge vào `main` để release, merge ngược vào `develop` để chống trôi lỗi.

## 2. Bối cảnh
- `main` đang chạy stable `v1.0.0`, phát hiện lỗi nghiêm trọng lộ dữ liệu người dùng.
- `develop` đang dở dang tính năng mới, không thể deploy ngay → không được fix thẳng trên `develop`.
- Giải pháp: tách `hotfix/v1.0.1` từ `main`, fix, release, rồi đồng bộ ngược về `develop`.

## 3. Các bước thực hiện

### Bước 1 — Chuẩn bị `main` và `develop`
```bash
git init -b main
echo "console.log('v1.0.0 stable');" > app.js
git add app.js && git commit -m "release v1.0.0"  # 70a7146
git tag -a v1.0.0 -m "Release 1.0.0"

git checkout -b develop
echo "// feature moi dang phat trien do (chua xong)" >> app.js
git add app.js && git commit -m "feat: dang phat trien tinh nang moi" # 69c516f
```

### Bước 2 — Tạo nhánh hotfix từ `main`
```bash
git checkout main
git checkout -b hotfix/v1.0.1
```

### Bước 3 — Sửa lỗi trên hotfix
```bash
# app.js trước fix:
# console.log('v1.0.0 stable');

# app.js sau fix:
# console.log('v1.0.0 stable'); // FIX: che chan du lieu nguoi dung

git add app.js
git commit -m "fix: chan lo hong lo du lieu nguoi dung"  # c0ead77
```

### Bước 4 — Gộp hotfix vào `main` + gắn tag release
```bash
git checkout main
git merge hotfix/v1.0.1
# Updating 70a7146..c0ead77 (Fast-forward)

git tag -a v1.0.1 -m "Release Hotfix 1.0.1"
```

### Bước 5 — Gộp ngược hotfix vào `develop` (bắt buộc)
```bash
git checkout develop
git merge hotfix/v1.0.1
```
Lần này có conflict vì `develop` và `hotfix` cùng sửa dòng đầu của `app.js`:
```
Auto-merging app.js
CONFLICT (content): Merge conflict in app.js
```
File conflict:
```text
<<<<<<< HEAD
console.log('v1.0.0 stable');
// feature moi dang phat trien do (chua xong)
=======
console.log('v1.0.0 stable'); // FIX: che chan du lieu nguoi dung
>>>>>>> hotfix/v1.0.1
```

Giải quyết bằng cách giữ cả hai: vừa có fix bảo mật, vừa giữ code tính năng dở dang:
```js
console.log('v1.0.0 stable'); // FIX: che chan du lieu nguoi dung
// feature moi dang phat trien do (chua xong)
```
```bash
git add app.js
git commit -m "Merge branch 'hotfix/v1.0.1' into develop"  # 0c1b5eb
```

### Bước 6 — Dọn dẹp nhánh hotfix
```bash
git branch -d hotfix/v1.0.1
# Deleted branch hotfix/v1.0.1 (was c0ead77).
```

## 4. Kiểm tra
```bash
git branch -a
```
```
* develop
  main
```

```bash
git tag
```
```
v1.0.0
v1.0.1
```

```bash
git log --graph --oneline --all --decorate
```
```
*   0c1b5eb (develop) Merge branch 'hotfix/v1.0.1' into develop
|\
| * c0ead77 (main, tag: v1.0.1) fix: chan lo hong lo du lieu nguoi dung
* | 69c516f feat: dang phat trien tinh nang moi
|/
* 70a7146 (tag: v1.0.0) release v1.0.0
```

```bash
git show v1.0.1 --stat
# tag v1.0.1 - Release Hotfix 1.0.1
# commit c0ead77 fix: chan lo hong lo du lieu nguoi dung
```

Nội dung file sau cùng:
- Trên `main` (`app.js`): `console.log('v1.0.0 stable'); // FIX: che chan du lieu nguoi dung`
- Trên `develop` (`app.js`): có thêm dòng `// feature moi dang phat trien do (chua xong)` + đã có FIX.

## 5. Sơ đồ Gitflow (ASCII)

```
        hotfix/v1.0.1
       /             \
      /               \
main *---*-------------*  (v1.0.1, fast-forward)
     |   |             |
     |   |   develop   *---*  (merge hotfix ngược về develop)
     |   |    /            \ /
     |   |   /              X  (resolve conflict: giữ FIX + feature dở dang)
     v1.0.0 *---------------*
           release      feat mới
```

Giải thích luồng:
1. `main (v1.0.0)` → tách `hotfix/v1.0.1`.
2. `hotfix` → merge vào `main` (fast-forward vì `main` không có commit mới) → tag `v1.0.1` để release.
3. `hotfix` → merge ngược vào `develop` (merge commit `0c1b5eb`, có xử lý conflict).
4. Xóa `hotfix/v1.0.1` sau khi hoàn tất.

> Ảnh chụp cần nộp: thay ASCII trên bằng ảnh `git log --graph --oneline --all --decorate` + ảnh `git branch -a` và `git tag`.

## 6. Đối chiếu yêu cầu
- [x] Nhánh `hotfix/v1.0.1` tách trực tiếp từ `main`.
- [x] Merge vào `main` + tag `v1.0.1` trỏ đúng commit fix trên `main`.
- [x] Merge ngược vào `develop`; lịch sử `develop` chứa commit fix (`c0ead77` là ancestor của `0c1b5eb`).
- [x] Dọn dẹp nhánh hotfix sau release.

## 7. Bài học Gitflow
- `main` luôn deploy được; `develop` là tích hợp tính năng tương lai.
- Hotfix luôn đi từ `main` và phải về cả 2 nơi (`main` + `develop`), nếu quên merge ngược sẽ bị hồi lỗi ở release sau.
- Tag annotated (`-a`) lưu người tạo, ngày, message — bắt buộc cho release chính thức.
