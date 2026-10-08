# Quy trinh phat trien va quan ly repository

## 1. Nguyen tac

- `main` la branch production-ready, khong phat trien truc tiep tren `main`.
- Moi feature, bugfix, hotfix hoac thay doi tai lieu phai co branch rieng.
- Moi thay doi vao `main` phai di qua Pull Request.
- Chi merge sau khi nguoi phu trach da test va xac nhan `OK`.
- Khong force-push, khong xoa `main`, khong commit secret.

Ap dung cho:

- `PanxiiPos/panxii-backend`
- `PanxiiPos/panxii-mobile`
- `PanxiiPos/panxii-docs`

## 2. Dat ten branch

Cu phap:

```text
<type>/<short-description>
```

Type duoc phep:

| Type | Khi dung | Vi du |
|---|---|---|
| `feature` | Capability moi | `feature/invoice-sync` |
| `fix` | Sua bug | `fix/tenant-filter-leak` |
| `hotfix` | Sua loi khan cap tren production | `hotfix/auth-token-expiry` |
| `refactor` | Doi cau truc, khong doi behavior | `refactor/result-contract` |
| `docs` | Thay doi tai lieu | `docs/update-api-contract` |
| `chore` | Bao tri dependency/tooling | `chore/update-dotnet-sdk` |

Quy tac:

- Dung lowercase va dau gach ngang.
- Mo ta ngan, co nghia, khong dung ten ca nhan.
- Khong dung `dev`, `test`, `new`, `changes` lam branch chung.
- Một feature chạm nhiều repo thì tạo branch cùng tên ở từng repo.

## 3. Bat dau feature

```powershell
# Cap nhat main
 git switch main
 git pull --ff-only origin main

# Tao branch tu main moi nhat
 git switch -c feature/invoice-sync
```

Lam viec va commit theo [commit_convention.md](commit_convention.md). Push branch:

```powershell
git push -u origin feature/invoice-sync
```

## 4. Pull Request

Mo PR tu branch feature vao `main` trong cung repository.

PR title dung Conventional Commits:

```text
feat(sync): add idempotent invoice synchronization
```

PR body toi thieu phai co:

- What: thay doi gi.
- Why: tai sao can thay doi.
- Test: da chay lenh/test nao.
- Risk: rui ro, migration, compatibility va rollback.
- Screenshots/log response neu la UI/API.

Khong merge khi:

- Chua test.
- Con comment chua xu ly.
- Test fail hoac build fail.
- Co thay doi ngoai scope ma khong giai thich.
- Co secret, PII, `.env`, key hoac database dump.

## 5. Quy trinh test va merge

1. Developer push branch feature.
2. Tao Pull Request vao `main`.
3. Chay test tuong ung va ghi ket qua trong PR.
4. Developer/owner test thu cong tren local, staging hoac device.
5. Chi khi test `OK` moi merge PR.
6. Merge theo quy dinh repository, uu tien Squash merge.
7. Xoa branch feature sau khi merge.
8. Cap nhat local va bat dau feature moi tu `main`.

Lenh sau merge:

```powershell
git switch main
git pull --ff-only origin main
git branch -d feature/invoice-sync
git push origin --delete feature/invoice-sync
```

## 6. Hotfix

Hotfix van phai di qua PR:

```powershell
git switch main
git pull --ff-only origin main
git switch -c hotfix/short-description
```

Hotfix phai co test regression va ghi ro pham vi anh huong trong PR.

## 7. Bao ve main tren GitHub

Ba repository da tao ruleset `Protect main` voi cac quy tac:

- Target branch: `main`.
- Bat buoc Pull Request truoc khi merge.
- Chan force-push.
- Chan xoa branch.
- Required approvals: `0`, de owner co the tu test va merge PR cua minh.

Luu y: GitHub hien thong bao ruleset cho repository private chi duoc enforce khi Organization `PanxiiPos` nang len goi GitHub Team. Truoc khi nang goi, quy trinh PR van la quy uoc bat buoc cua team nhung GitHub chua chan ky thuat viec push truc tiep. Khong push truc tiep vao `main` du quy tac da tao.

## 8. Quy uoc nhanh cho moi task

```text
1. Doc business rule va architecture.
2. Tao branch tu main moi nhat.
3. Implement dung scope.
4. Commit theo Conventional Commits.
5. Push branch va tao PR.
6. Test va ghi ket qua.
7. Chi merge khi test OK.
8. Xoa branch sau merge.
```
