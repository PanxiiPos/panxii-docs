# Quy uoc Commit Panxii

## 1. Muc tieu

Moi commit phai de doc, truy vet duoc va co the rollback. Commit la don vi thay doi nho nhat hop ly, phuc vu code review, CI/CD va release theo Git SHA.

Quy uoc nay ap dung cho:

- `panxii-backend`
- `panxii-mobile`
- `panxii-docs`

## 2. Cu phap commit

Dung Conventional Commits:

```text
<type>(<scope>): <subject>

<body>

<footer>
```

Chi `subject` la bat buoc. `body` va `footer` dung khi can giai thich hoac lien ket issue.

Vi du:

```text
feat(invoice): add idempotent invoice sync command

Store the operation id with the invoice transaction so retries return
 the original result instead of creating a duplicate invoice.

Refs: #123
```

## 3. Type duoc phep

| Type | Su dung cho |
|---|---|
| `feat` | Them capability hoac behavior moi cho nguoi dung/he thong |
| `fix` | Sua bug lam sai behavior hien tai |
| `refactor` | Doi cau truc code, khong doi behavior |
| `perf` | Cai thien hieu nang ma khong doi contract |
| `test` | Them, sua hoac bo sung test |
| `docs` | Sua tai lieu, OpenAPI, ADR hoac comment huong dan |
| `build` | Sua SDK, package, project file hoac build configuration |
| `ci` | Sua pipeline, workflow hoac quality gate |
| `chore` | Bao tri ky thuat khong thuoc cac nhom tren |
| `revert` | Hoan tac mot commit truoc do |

Khong dung `update`, `change`, `misc` hoac `other` khi mot type cu the phu hop hon.

## 4. Scope

Scope phai ngan gon va chi ro khu vuc bi anh huong. Scope goi y:

| Scope | Pham vi |
|---|---|
| `domain` | Entity, value object, invariant, domain event |
| `application` | Command, query, handler, validator, Result |
| `api` | Controller, middleware, DTO, OpenAPI, error envelope |
| `infra` | EF Core, MySQL, RabbitMQ, SignalR, external adapter |
| `auth` | JWT, RBAC, device session, revoke |
| `tenant` | Tenant context, isolation, global query filter |
| `invoice` | Invoice, reverse/correction, payment state |
| `sync` | Offline outbox, idempotency, cursor, pull |
| `mobile` | Flutter UI, BLoC, local database, localization |
| `docs` | Tai lieu dung chung trong `panxii-docs` |
| `ci` | GitHub Actions, quality gate, release automation |
| `deps` | Dependency/package update |

Neu thay doi nhieu scope khong the tach hop ly, bo scope va mo ta pham vi trong body.

## 5. Quy tac viet subject

- Viet tieng Anh, dong tu o dang menh lenh: `add`, `fix`, `remove`, `update`.
- Viet thuong sau dau `:`; khong dat dau cham o cuoi.
- Toi da 72 ky tu; uu tien ngan hon 60 ky tu.
- Khong viet chung chung: `fix bug`, `update code`, `changes`.
- Khong dua issue number vao subject neu da co footer.
- Mot commit chi nen co mot muc dich thay doi.

Subject tot:

```text
feat(tenant): enforce tenant filter on product queries
fix(sync): preserve operation id across retry
refactor(api): move error mapping to middleware
docs(docs): add commit convention
```

Subject khong dat:

```text
update code
fix bug
FEAT: Add stuff.
```

## 6. Body va footer

Body cach subject mot dong trong, moi dong toi da 72 ky tu khi co the. Noi ro:

1. Tai sao thay doi nay can thiet.
2. Behavior nao da thay doi.
3. Trade-off, migration hoac compatibility nao can luu y.

Footer dung cho:

- `Refs: #123`: lien ket cong viec khong dong issue.
- `Closes: #123`: commit/PR hoan tat issue.
- `BREAKING CHANGE: ...`: thay doi khong tuong thich.
- `Reviewed-by: <name>`: chi dung khi quy trinh review yeu cau.

Vi du breaking change:

```text
feat(api)!: version invoice sync endpoint

BREAKING CHANGE: clients must send operationId in the v2 request body.
```

Dau `!` la tuy chon de nhan manh; footer `BREAKING CHANGE` van phai neu ro tac dong va cach migrate.

## 7. Quy tac theo loai thay doi

### Business rule va domain

- Doi invariant quan trong phai co test domain trong cung commit/PR.
- Khong sua invoice da sync bang update/delete; dung reverse/correction theo business rule.
- Neu thay doi business rule da duyet, cap nhat ADR hoac tai lieu lien quan trong cung PR.

### API va database

- OpenAPI, error catalog, DTO, mapping mobile va contract test phai cap nhat cung PR.
- Migration phai tach commit hop ly va ghi ro expand/contract, backward compatibility va rollback.
- Khong commit `.env`, password that, token, private key, database dump hoac du lieu PII.

### Offline sync

- Thay doi idempotency, outbox, cursor hoac retry phai co test timeout/retry va duplicate protection.
- Ghi ro neu thay doi anh huong mobile version cu.

### Tai lieu

- Dung `docs(scope): ...` cho tai lieu trong repo ung dung.
- Dung `docs(docs): ...` cho `panxii-docs`.
- Sơ do Mermaid, OpenAPI va vi du phai validate neu thay doi contract.

## 8. Commit va pull request

- Khong commit truc tiep vao `main` cho feature/bugfix.
- Tao feature branch moi tu nhanh moi nhat ma feature can lam nen; khong mac dinh tao tu `main` neu feature phu thuoc code chua merge.
- Co the xep tang feature branch: feature moi tao tu feature branch truoc do khi can dung code cua feature do. Ghi ro nhanh nen/dependency trong PR.
- Feature cu hon co the merge vao `main` ngay khi da on dinh va pass cac quality gate. Feature moi phu thuoc van tiep tuc tren nhanh cua minh; cap nhat/rebase voi `main` khi can va chi merge sau khi da fix, test va on dinh.
- Mot PR nen tap trung mot muc tieu va co commit co y nghia.
- Rebase/squash theo quy dinh cua PR; khong dung merge commit neu khong can luu lich su nhanh.
- Trước khi push, chay test phu hop voi scope thay doi.
- PR title dung cung cu phap Conventional Commits.
- PR body phai co summary, test da chay, migration/rollback va risk neu co.

Tao nhanh feature tu nhanh nen da cap nhat:

```powershell
git fetch origin
git switch <base-branch>
git pull --ff-only
git switch -c feature/<feature-name>
```

## 9. Checklist truoc commit

- [ ] Commit dung `type(scope): subject`.
- [ ] Subject la dong tu menh lenh, viet thuong, khong qua 72 ky tu.
- [ ] Commit khong chua secret, PII, `.env`, key hoac file generated.
- [ ] Thay doi business rule co test tuong ung.
- [ ] API/OpenAPI/error code/mobile mapping da dong bo neu lien quan.
- [ ] Database migration va rollback plan da duoc ghi ro neu lien quan.
- [ ] Test, formatter va analyzer phu hop da pass.
- [ ] Commit khong gom thay doi khong lien quan.

## 10. Lenh tham khao

```powershell
# Kiem tra thay doi
 git diff --check
 git status

# Backend
 dotnet build Panxii.sln
 dotnet test Panxii.sln

# Tao commit
 git add <files>
 git commit -m "feat(invoice): add invoice sync command"
```
