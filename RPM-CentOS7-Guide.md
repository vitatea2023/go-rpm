# RPM 打包通用指南

## 概述
本指南提供了将二进制程序打包为 RPM 的通用方法，确保安装后能自启动、开机自启动，并兼容 CentOS 7 等系统。

## 核心要求
- ✅ 安装后自动启动服务
- ✅ 开机自启动
- ✅ CentOS 7 兼容性（使用 gzip 压缩）
- ✅ systemd 服务集成
- ✅ 完整的生命周期管理（安装/卸载/升级）

## 环境准备
### 必需工具包
```bash
dnf install -y rpm-build systemd-rpm-macros rpmdevtools
```

### 创建构建目录
```bash
mkdir -p rpmwork/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
```

## 打包步骤

### 1. 准备二进制文件
```bash
# 设置执行权限
chmod +x your_binary_file

# 复制到 SOURCES 目录
cp your_binary_file rpmwork/SOURCES/
```

### 2. 创建 RPM Spec 文件
在 `rpmwork/SPECS/package.spec` 创建：

```spec
Name:           your-package-name
Version:        1.0.0
Release:        1%{?dist}
Summary:        Your service description

License:        Proprietary
URL:            https://your-domain.com
Source0:        your_binary_file

BuildArch:      x86_64
BuildRequires:  systemd-rpm-macros
%global debug_package %{nil}
Requires(post): systemd
Requires(preun): systemd
Requires(postun): systemd

%description
Your service description here.

%prep
%setup -q -c -T
cp %{SOURCE0} .

%build
# Static binary, no build needed

%install
# Create directories
mkdir -p %{buildroot}/usr/local/your-service
mkdir -p %{buildroot}%{_unitdir}

# Install binary
install -m 755 your_binary_file %{buildroot}/usr/local/your-service/your_binary_file

# Create systemd service file
cat > %{buildroot}%{_unitdir}/your-service.service << 'EOF'
[Unit]
Description=Your Service Description
After=network.target

[Service]
ExecStart=/usr/local/your-service/your_binary_file --your-args
WorkingDirectory=/usr/local/your-service
Restart=always
RestartSec=5
User=root
Group=root

[Install]
WantedBy=multi-user.target
EOF

%files
/usr/local/your-service/your_binary_file
%{_unitdir}/your-service.service

%post
%systemd_post your-service.service
# Enable and start the service
systemctl enable your-service.service >/dev/null 2>&1 || :
systemctl start your-service.service >/dev/null 2>&1 || :

%preun
%systemd_preun your-service.service

%postun
%systemd_postun_with_restart your-service.service

%changelog
* $(date +'%a %b %d %Y') System Administrator <admin@example.com> - 1.0.0-1
- Initial package
```

### 3. 构建 RPM 包
```bash
cd rpmwork
rpmbuild --define "_topdir $(pwd)" \
         --define "_binary_payload w2.gzdio" \
         --target x86_64 \
         --define "dist .el7" \
         -ba SPECS/your-package.spec
```

### 4. 验证构建结果
```bash
# 检查压缩格式（应显示 gzip）
rpm -qp --queryformat '%{PAYLOADCOMPRESSOR}\n' RPMS/x86_64/your-package-*.rpm

# 查看包内容
rpm -qpl RPMS/x86_64/your-package-*.rpm

# 查看包信息
rpm -qpi RPMS/x86_64/your-package-*.rpm
```

## Spec 文件关键配置说明

### systemd 集成
```spec
BuildRequires:  systemd-rpm-macros
Requires(post): systemd
Requires(preun): systemd
Requires(postun): systemd
```

### 自启动配置（关键）
```spec
%post
%systemd_post your-service.service
# 安装后立即启用并启动服务
systemctl enable your-service.service >/dev/null 2>&1 || :
systemctl start your-service.service >/dev/null 2>&1 || :
```

### CentOS 7 兼容性
- 使用 `--define "_binary_payload w2.gzdio"` 强制 gzip 压缩
- 使用 `--define "dist .el7"` 设置发行版标识

### 调试信息禁用
```spec
%global debug_package %{nil}
```
静态编译的二进制文件无需调试包。

## systemd 服务文件模板
```ini
[Unit]
Description=Your Service Description
After=network.target
# 可选：添加依赖
# Requires=network-online.target
# After=network-online.target

[Service]
ExecStart=/usr/local/your-service/your_binary_file --your-args
WorkingDirectory=/usr/local/your-service
Restart=always
RestartSec=5
User=root
Group=root
# 可选：环境变量
# Environment=YOUR_VAR=value
# 可选：资源限制
# LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## 安装和验证
### 安装 RPM 包
```bash
rpm -ivh your-package-*.rpm
```

### 验证服务状态
```bash
# 检查服务状态
systemctl status your-service.service

# 检查开机自启状态（应显示 enabled）
systemctl is-enabled your-service.service

# 查看服务日志
journalctl -u your-service.service -f
```

### 预期结果
- 服务状态：`active (running)`
- 开机自启：`enabled`
- 服务正常运行并输出日志

## 常见问题排查

### 1. RPM 压缩格式不兼容
**症状**：`rpmlib(PayloadIsZstd) <= 5.4.18-1 is needed`
**解决**：使用 `--define "_binary_payload w2.gzdio"` 重新构建

### 2. 服务启动失败
**排查步骤**：
```bash
# 查看详细错误日志
journalctl -u your-service.service -n 50

# 手动测试二进制文件
/usr/local/your-service/your_binary_file --your-args

# 检查文件权限
ls -la /usr/local/your-service/
```

### 3. 服务未自启动
**检查项**：
- 确认 spec 文件中的 `%post` 脚本包含 `systemctl enable`
- 确认 systemd 服务文件的 `[Install]` 部分正确

### 4. 网络依赖问题
**解决**：在服务文件中添加网络依赖
```ini
[Unit]
After=network-online.target
Wants=network-online.target
```

## 高级配置选项

### 用户和权限
```spec
# 创建专用用户（可选）
%pre
getent group your-service >/dev/null || groupadd -r your-service
getent passwd your-service >/dev/null || \
    useradd -r -g your-service -d /usr/local/your-service -s /sbin/nologin \
    -c "Your Service User" your-service
```

### 配置文件支持
```spec
# 在 %install 中添加配置目录
mkdir -p %{buildroot}/etc/your-service

# 在 %files 中添加配置文件
%config(noreplace) /etc/your-service/config.yaml
```

### 多架构支持
```spec
# 移除 BuildArch 限制，支持多架构
# BuildArch: x86_64
```

## 最佳实践总结
1. **总是使用 gzip 压缩**确保 CentOS 7 兼容性
2. **完整的 systemd 集成**包含所有生命周期钩子
3. **%post 脚本中立即启用并启动服务**确保安装后可用
4. **静态二进制文件禁用调试包**减少包大小
5. **合理的重启策略**服务异常时自动重启
6. **详细的服务描述和变更日志**便于维护

---

**注意**：根据实际需求调整服务名称、路径、启动参数等配置项。


