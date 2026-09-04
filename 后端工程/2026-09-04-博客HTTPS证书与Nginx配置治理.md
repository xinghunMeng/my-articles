# 博客 alleria.cn 的 Let's Encrypt 部署与一处 nginx 配置漂移修复
category: 后端工程

## 问题背景

博客系统（Node.js Express + Markdown）部署在腾讯云 Ubuntu 实例，对外提供 blog.alleria.cn，域名挂在简历上。此前仅提供 HTTP 服务，存在两个问题：浏览器标记为不安全、缺少传输层加密。目标是在不影响在运服务的前提下完成 HTTPS 接入，并消除一处历史配置隐患。

## 技术方案

使用 certbot 的 nginx 插件完成证书签发与反代改写：

```bash
sudo certbot --nginx -d blog.alleria.cn --redirect -n --agree-tos
```

该命令签发 ECDSA 证书（路径 /etc/letsencrypt/live/blog.alleria.cn/，有效期 89 天），并自动改写 /etc/nginx/sites-enabled/blog，加入 443 ssl 监听与 80→301 跳转。续期由 certbot.timer 负责（每日两次执行 certbot renew，--nginx 插件自带 deploy hook，在续期后自动 reload nginx），与同机 getoffer.alleria.cn 共用同一套机制。

外网验证结果：HTTP 请求返回 301、HTTPS 返回 200、HTTPS 下访问不存在的 /server.js 返回 404，均符合预期。

## 踩坑记录

历史配置写入时，heredoc 把变量 `$proxy_add_x_forwarded_for` 误写为 `$proxy_add_x_forward_for`（少 ed）。该变量未定义，导致 on-disk 配置执行 nginx -t 时报 [emerg]；但当时 running nginx 仍使用内存中的旧配置，外网正常，隐患被掩盖，直到某次重启才会触发故障。

本次重连服务器后执行 `sudo nginx -t`（syntax is ok / test is successful）与 `sudo nginx -s reload`（signal process started），确认 on-disk 与 running 配置均已修正并一致。

## 工程复盘

1. 配置漂移（running 与 on-disk 不一致）是最隐蔽的运维风险，必须执行“改完即 nginx -t、随即 reload 验证”，不能只靠外网自查。
2. 同机多站点 ssl 配置可并存：certbot 默认走 options-ssl-nginx.conf include，另一站点手写 ssl_ciphers 互不影响。
3. 对无害 warn（如本例 ssl_stapling 在无 OCSP URL 时的提示）遵循最小改动原则，不主动折腾。
4. 把整套流程沉淀为 https-cert-setup skill：六步工作流（收集参数→环境检查→签发→验证→续期确认→汇报）+ 踩坑库 + 无 sshpass 的密码 SSH 模板，后续其它域名可直接复用。

相关：blog.alleria.cn

