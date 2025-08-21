# Tích hợp SSH với Git — Hướng dẫn Từng Bước (Markdown)

> Mục tiêu: Thiết lập SSH để làm việc với Git (GitHub/GitLab/Bitbucket) không cần nhập username/password, bảo mật hơn và thuận tiện cho cả môi trường cá nhân lẫn doanh nghiệp.

---

## 1) Vì sao nên dùng SSH thay vì HTTPS?

- **Không cần mật khẩu mỗi lần `git pull/push`**: Xác thực bằng khóa công khai/cá nhân.
- **Bảo mật cao**: Khóa riêng (private key) nằm trên thiết bị; có thể thêm **passphrase**.
- **Quản lý đa tài khoản**: Tạo nhiều cặp key và điều hướng bằng `~/.ssh/config`.
- **Tích hợp tốt trong CI/CD**: Dùng deploy key/robot account cho pipeline.

> Tóm lại: dùng SSH giúp trải nghiệm mượt mà, bảo mật và dễ tự động hóa.

---

## 2) Yêu cầu hệ thống

- **Git** (2.39+ khuyến nghị) và **OpenSSH** có sẵn trên macOS/Linux; trên Windows dùng **Git for Windows** hoặc **OpenSSH Client**.
- Tài khoản trên Git provider: **GitHub**, **GitLab** hoặc **Bitbucket**.
- Quyền truy cập repo (đọc/ghi) phù hợp.

---

## 3) Kiểm tra nhanh môi trường

```bash
# Kiểm tra Git
git --version

# Kiểm tra OpenSSH client
ssh -V

# Thử kết nối đến GitHub (ví dụ)
ssh -T git@github.com
```

- Nếu thấy `Permission denied (publickey)` → chưa cài key.
- Nếu thấy `Hi <username>! You've successfully authenticated...` → SSH đã sẵn sàng.

> Lưu ý: với GitLab dùng `ssh -T git@gitlab.com`, với Bitbucket dùng `ssh -T git@bitbucket.org`.

---

## 4) Tạo SSH Key mới

### Thuật toán khuyến nghị
- **Ed25519** (nhanh, bảo mật, key ngắn) — ưu tiên.
- **RSA 4096-bit** chỉ khi máy/thiết bị cũ không hỗ trợ Ed25519.

### Lệnh tạo key (macOS/Linux/Windows qua Git Bash)

```bash
# Ed25519 (khuyến nghị)
ssh-keygen -t ed25519 -C "email_github@example.com"

# RSA (fallback nếu cần)
# ssh-keygen -t rsa -b 4096 -C "email_github@example.com"
```

**Giải thích tham số**:
- `-t`: loại thuật toán (ed25519/rsa).
- `-b 4096`: số bit (chỉ áp dụng cho RSA).
- `-C`: comment (thường dùng email để dễ nhận diện trên Git provider).

**Trả lời các prompt**:
- **File lưu key**: Nhấn **Enter** để dùng mặc định (`~/.ssh/id_ed25519`).
- **Passphrase**: Nên **đặt** (tăng bảo mật). Có thể để trống nếu ưu tiên tiện dụng.

**Vị trí file**:
- Private key: `~/.ssh/id_ed25519`
- Public key: `~/.ssh/id_ed25519.pub`

> Windows (PowerShell) tương tự nếu đã cài OpenSSH Client; đường dẫn nằm tại `C:\Users\<User>\.ssh`.

---

## 5) Nạp key vào ssh-agent (cache passphrase)

### macOS / Linux (bash/zsh)
```bash
# Khởi động agent
eval "$(ssh-agent -s)"

# Thêm private key vào agent
ssh-add ~/.ssh/id_ed25519
```

### Windows 10/11 (PowerShell)
```powershell
# Bật dịch vụ ssh-agent và cho tự động khởi động
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# Thêm private key
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

> Nếu thấy lỗi "Could not open a connection to your authentication agent" trên bash, hãy chạy `eval "$(ssh-agent -s)"` trước, sau đó `ssh-add`.

---

## 6) Đăng ký Public Key lên Git Provider

### 6.1 GitHub
1. Sao chép **public key**:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
2. Vào **GitHub → Settings → SSH and GPG Keys → New SSH key**.
3. Dán nội dung public key, đặt tên (ví dụ: `MacBook Pro - 2025`), nhấn **Add SSH key**.
4. Kiểm tra:
   ```bash
   ssh -T git@github.com
   ```

### 6.2 GitLab
1. Sao chép public key.
2. **GitLab → Preferences → SSH Keys → Add new key**.
3. Dán key, đặt `Title`, và **Add key**.
4. Kiểm tra:
   ```bash
   ssh -T git@gitlab.com
   ```

### 6.3 Bitbucket
1. Sao chép public key.
2. **Bitbucket → Personal settings → SSH keys → Add key**.
3. Dán key, lưu.
4. Kiểm tra:
   ```bash
   ssh -T git@bitbucket.org
   ```

---

## 7) Dùng SSH với Repository

### 7.1 Clone bằng SSH
```bash
git clone git@github.com:<username>/<repo>.git
# hoặc GitLab
# git clone git@gitlab.com:<group>/<repo>.git
# hoặc Bitbucket
# git clone git@bitbucket.org:<workspace>/<repo>.git
```

### 7.2 Đổi remote từ HTTPS → SSH (repo đã clone)
```bash
# Xem remote hiện tại
git remote -v

# Đổi sang SSH (GitHub)
git remote set-url origin git@github.com:<username>/<repo>.git

# Kiểm tra lại
git remote -v
```

> Từ đây có thể `git pull`, `git push` mà không cần nhập mật khẩu.

---

## 8) Quản lý **nhiều tài khoản** hoặc **nhiều Git provider**

### 8.1 Tạo nhiều cặp key
```bash
# Cá nhân
ssh-keygen -t ed25519 -C "your_personal_email@example.com" -f ~/.ssh/id_ed25519_personal

# Công ty
ssh-keygen -t ed25519 -C "your_work_email@company.com" -f ~/.ssh/id_ed25519_work
```

Thêm cả hai key vào agent:
```bash
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

### 8.2 Cấu hình `~/.ssh/config`
```sshconfig
# Mặc định cho GitHub (tài khoản cá nhân)
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  IdentitiesOnly yes

# Alias cho GitHub công ty
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
```

**Cách dùng**:
- Repo cá nhân clone như bình thường:
  ```bash
  git clone git@github.com:<personal-username>/<repo>.git
  ```
- Repo công ty clone với **alias** `github-work`:
  ```bash
  git clone git@github-work:<company-org>/<repo>.git
  ```

> Tương tự với GitLab/Bitbucket: đặt `Host gitlab-work` hoặc `bitbucket-work` và `HostName` tương ứng (`gitlab.com`, `bitbucket.org`).

### 8.3 Tách email/username Git theo thư mục dự án
Sử dụng **conditional include** trong `~/.gitconfig`:

```ini
# ~/.gitconfig (toàn cục)
[user]
  name = Your Personal Name
  email = your_personal_email@example.com

[includeIf "gitdir:~/work/"]
  path = ~/.gitconfig-work
```

Và trong `~/.gitconfig-work`:
```ini
[user]
  name = Your Work Name
  email = your_work_email@company.com
```

> Khi các repo công ty để dưới `~/work/`, Git sẽ tự dùng email công ty.

---

## 9) Kiểm tra & Xác minh kết nối

```bash
# Kiểm tra host GitHub
ssh -T git@github.com

# In chi tiết quá trình handshake (debug)
ssh -vT git@github.com
```

Kết quả thành công (GitHub):
```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## 10) Troubleshooting — Lỗi thường gặp và cách xử lý

### 10.1 `Permission denied (publickey)`
**Nguyên nhân**: Key chưa add vào agent, key sai, hoặc public key chưa add lên Git provider.

**Khắc phục**:
```bash
# Kiểm tra key đã có trong agent?
ssh-add -l

# Nếu chưa có, thêm lại
ssh-add ~/.ssh/id_ed25519

# So khớp public key với Git provider
cat ~/.ssh/id_ed25519.pub
```

### 10.2 `Host key verification failed`
**Nguyên nhân**: Chưa tin cậy fingerprint của host, file `known_hosts` lỗi.

**Khắc phục**:
```bash
# Xóa fingerprint cũ rồi kết nối lại
ssh-keygen -R github.com
ssh -T git@github.com
```
Xác nhận fingerprint mới khi được hỏi.

### 10.3 `Bad permissions` cho file key
**Nguyên nhân**: Quyền file private key quá rộng.

**Khắc phục (Linux/macOS)**:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```
Windows không áp dụng `chmod` như Unix, nhưng đảm bảo file chỉ thuộc user hiện tại.

### 10.4 Dùng sai **Host/Alias** trong `~/.ssh/config`
**Triệu chứng**: `Could not resolve hostname github-work: Name or service not known`.

**Khắc phục**: Đảm bảo có block `Host github-work` và `HostName github.com`, clone bằng `git@github-work:org/repo.git`.

### 10.5 `Agent admitted failure to sign using the key`
**Khắc phục**: Khởi động lại `ssh-agent` và add key lại.
```bash
eval "$(ssh-agent -s)"
ssh-add -D
ssh-add ~/.ssh/id_ed25519
```

### 10.6 Repo private báo `Repository not found`
**Nguyên nhân**: Key không gắn với tài khoản có quyền, hoặc remote URL sai.

**Khắc phục**:
- Xác thực bằng `ssh -T` đến đúng host.
- Kiểm tra `git remote -v` và đảm bảo đúng `org/repo`.

### 10.7 Qua proxy/doanh nghiệp
- Dùng `ProxyCommand` trong `~/.ssh/config` hoặc cổng SOCKS/HTTP (tuỳ chính sách công ty).
- Có thể cần trao đổi với IT để whitelist domain `github.com`/`gitlab.com`.

---

## 11) Thực hành bảo mật tốt (Best Practices)

- **Một thiết bị → một cặp key**, dễ thu hồi khi mất máy.
- **Dùng passphrase** cho private key, bật `ssh-agent` để cache.
- **Xoá key cũ/không dùng** trên Git provider.
- **Xoay vòng (rotate) định kỳ**: tạo key mới, add lên, test OK rồi revoke key cũ.
- **Sao lưu tối thiểu**: Nếu buộc phải backup private key, mã hoá và lưu an toàn (không commit vào repo!).
- **Cân nhắc khoá bảo mật phần cứng** (FIDO2/WebAuthn, dạng `sk-ssh-ed25519@openssh.com`) cho môi trường yêu cầu bảo mật cao.

---

## 12) Dùng SSH trong CI/CD (tuỳ chọn)

- **Deploy key (GitHub/GitLab/Bitbucket)**: key gắn với 1 repo, cấp quyền chỉ-đọc/đọc-ghi.
- **Robot/Service account**: tạo user kỹ thuật với key riêng để pipeline có thể `clone/push` theo phạm vi cần thiết.
- Trong pipeline, nạp private key vào `ssh-agent` và thiết lập `known_hosts`.

Ví dụ skeleton (Linux runner):
```bash
# Khởi tạo SSH trong CI
eval "$(ssh-agent -s)"
ssh-add <(echo "$PRIVATE_KEY")
mkdir -p ~/.ssh
ssh-keyscan -H github.com >> ~/.ssh/known_hosts

git clone git@github.com:org/repo.git
```

> Lưu ý: `PRIVATE_KEY` nên cấp qua **secret manager** của nền tảng CI/CD.

---

## 13) Checklist nhanh

- [ ] Cài Git + OpenSSH.
- [ ] Tạo key: `ssh-keygen -t ed25519 -C "email"`.
- [ ] Add vào agent: `ssh-add ~/.ssh/id_ed25519`.
- [ ] Thêm public key lên Git provider.
- [ ] Kiểm tra: `ssh -T git@github.com`.
- [ ] Clone/đổi remote sang SSH.
- [ ] (Tùy chọn) Cấu hình `~/.ssh/config` cho đa tài khoản.

---

## 14) FAQ

**Q1: Có thể dùng chung một key cho nhiều máy không?**  
A: Không khuyến khích. Nên tạo **một key/mỗi thiết bị** để dễ thu hồi nếu mất.

**Q2: HTTPS vẫn dùng được chứ?**  
A: Được, nhưng sẽ cần token/mật khẩu. SSH tiện và bảo mật hơn cho daily workflow.

**Q3: Dùng Windows thì nên tạo key ở đâu?**  
A: Dùng **Git Bash** hoặc **PowerShell** với OpenSSH Client. Key mặc định nằm ở `C:\Users\<User>\.ssh`.

**Q4: Em muốn commit ký số (GPG) thì liên quan SSH không?**  
A: Không. SSH dùng cho transport; ký commit dùng GPG/SSH signing riêng. Có thể cấu hình song song.

**Q5: Lỡ commit private key lên repo?**  
A: Xoá ngay khỏi repo (rewrite history nếu cần), **revoke key** trên Git provider và tạo key mới.

---

## 15) Phụ lục — Mẫu `~/.ssh/config`

```sshconfig
# ======= DEFAULT (Personal GitHub) =======
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  AddKeysToAgent yes

# ======= WORK (GitHub Enterprise alias) =======
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
  AddKeysToAgent yes

# ======= GitLab =======
Host gitlab.com
  HostName gitlab.com
  User git
  IdentityFile ~/.ssh/id_ed25519_gitlab
  IdentitiesOnly yes

# Optional: Proxy
# Host github.com
#   ProxyCommand nc -X connect -x proxy.company.com:8080 %h %p
```

---

## 16) Phụ lục — Lệnh nhanh (Cheat Sheet)

```bash
# Tạo key
ssh-keygen -t ed25519 -C "email@example.com"

# Khởi động agent & add key (Linux/macOS)
eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519

# Windows (PowerShell)
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519

# Test kết nối
ssh -T git@github.com

# Clone bằng SSH
git clone git@github.com:username/repo.git

# Đổi remote sang SSH
git remote set-url origin git@github.com:username/repo.git

# Debug SSH
essh -vT git@github.com
```

---

## 17) Tổng kết & Quan điểm

- **SSH là tiêu chuẩn vàng** cho làm việc với Git hằng ngày: tiện lợi, bảo mật và linh hoạt cho đa tài khoản.
- **Ed25519** hiện là lựa chọn tối ưu; chỉ fallback RSA khi bắt buộc.
- Thiết lập `~/.ssh/config` giúp workflow rõ ràng, tránh nhầm lẫn khi dùng nhiều tài khoản hoặc nhà cung cấp.
- Trong tổ chức, ưu tiên **deploy key/robot account**, quản trị **quyền tối thiểu**, và **xoay vòng key** theo chu kỳ để giảm rủi ro.

> Khuyến nghị: chuẩn hoá quy trình tạo/thu hồi key theo thiết bị, kèm checklist on/off-boarding để đảm bảo tuân thủ và an toàn lâu dài.
