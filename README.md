# Ansible Playbook สำหรับตั้งค่า EC2 ให้เชื่อถือ SSH Key จาก HashiCorp Vault (SSH CA)

Playbook นี้ใช้สำหรับ Configure เครื่อง EC2 (รองรับทั้ง Ubuntu/Debian และ Amazon Linux/RHEL/CentOS) เพื่อให้เชื่อถือ SSH Certificate Authority (CA) จาก HashiCorp Vault ทำให้สามารถ SSH เข้าเครื่อง EC2 ด้วย Certificate ที่เซ็นต์จาก Vault ได้โดยไม่ต้องกระจาย Public Key ไปไว้ที่ `~/.ssh/authorized_keys` ของแต่ละเครื่อง

---

## โครงสร้างโปรเจกต์

```text
.
├── ansible.cfg              # ตั้งค่าเริ่มต้นของ Ansible
├── inventory.ini            # กำหนด Host EC2 และ SSH Connection
├── group_vars/
│   └── all.yml              # ตัวแปรการตั้งค่า Vault และ SSHD
├── files/
│   └── vault_ca.pub         # ไฟล์ CA Key (กรณีเลือกดึงแบบ local file)
├── playbook.yml             # Playbook หลักสำหรับรัน Task
└── README.md
```

---

## 1. วิธีตั้งค่าตัวแปรใน [group_vars/all.yml](file:///Users/user/VaultkeyTrust/group_vars/all.yml)

เลือกวิธีการนำเข้า Vault SSH CA Public Key ได้ 3 รูปแบบตามความสะดวก:

### แบบที่ 1: ดึงจาก Vault API โดยตรง (แนะนำ)
```yaml
vault_ca_source_type: "api"
vault_addr: "https://vault.your-domain.com:8200"
vault_ssh_mount: "ssh" # mount path ของ SSH secrets engine
vault_validate_certs: true
```

### แบบที่ 2: ระบุ Key String ตรงๆ
```yaml
vault_ca_source_type: "string"
vault_ca_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG... vault-ca"
```

### แบบที่ 3: ระบุจากไฟล์ Local
วาง public key ไว้ที่ `files/vault_ca.pub` แล้วตั้งค่า:
```yaml
vault_ca_source_type: "file"
vault_ca_local_file: "files/vault_ca.pub"
```

---

## 2. วิธีตั้งค่า [inventory.ini](file:///Users/user/VaultkeyTrust/inventory.ini)

แก้ไขไฟล์ `inventory.ini` ให้ชี้ไปยัง EC2 instance ของคุณ:

```ini
[ec2_servers]
ec2-app-01 ansible_host=54.x.x.x ansible_user=ubuntu
# ec2-app-02 ansible_host=54.x.x.y ansible_user=ec2-user

[ec2_servers:vars]
# SSH Private Key สำหรับการเข้าถึง EC2 ครั้งแรกเพื่อรัน Ansible
ansible_ssh_private_key_file=~/.ssh/your-aws-ec2-key.pem
```

---

## 3. การรัน Playbook

ทดสอบ Syntax และ Dry-run (Check mode):
```bash
ansible-playbook -i inventory.ini playbook.yml --check
```

ยิงคำสั่งเพื่อตั้งค่าจริงไปยัง EC2:
```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## 4. Playbook ทำอะไรบ้างบน EC2?

1. **ตรวจสอบ OS**: เลือก service name (`ssh` สำหรับ Ubuntu/Debian หรือ `sshd` สำหรับ Amazon Linux/RHEL)
2. **ดึงและวาง CA Key**: บันทึก Public Key ของ Vault ไว้ที่ `/etc/ssh/trusted-user-ca-keys.pem` (สิทธิ์ `0644 root:root`)
3. **แก้ไข `/etc/ssh/sshd_config`**:
   - เปิดใช้งาน `PubkeyAuthentication yes`
   - กำหนด `TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem`
   - มีการ validate config ด้วย `sshd -t` ก่อนบันทึกเสมอ เพื่อป้องกันไม่ให้ SSH หลุด
4. **ตั้งค่า Principals (ตัวเลือกเสริม)**: จัดการ mapping สิทธิ์ใน `/etc/ssh/auth_principals/`
5. **Reload SSH**: ทำการ Reload SSH daemon โดยไม่ตัด connection ปัจจุบัน

---

## 5. วิธีทดสอบ Login เข้า EC2 ด้วย Vault SSH Cert

1. **สร้าง Client Key Pair** (ถ้ายังไม่มี):
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_vault -C "my-vault-key"
   ```

2. **ขอให้ Vault เซ็นต์ Public Key**:
   ```bash
   vault write -field=signed_key ssh/sign/my-role \
     public_key=@$HOME/.ssh/id_vault.pub \
     valid_principals="ubuntu,ec2-user" > ~/.ssh/id_vault-cert.pub
   ```

3. **ตรวจสอบรายละเอียด Certificate ที่ได้**:
   ```bash
   ssh-keygen -Lf ~/.ssh/id_vault-cert.pub
   ```

4. **SSH เข้า EC2 ด้วย Certificate**:
   ```bash
   ssh -i ~/.ssh/id_vault -i ~/.ssh/id_vault-cert.pub ubuntu@<EC2_IP>
   ```
