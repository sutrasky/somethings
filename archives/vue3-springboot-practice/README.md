# Vue3 + Spring Boot 练习项目归档

原样保存的练习项目 / 教程压缩包，2026-11-26 从手机附件导入。不解压、不改内容，用于留档和换机迁移。

| 文件 | 大小 | SHA256（前 12 位） | 内容 |
| --- | --- | --- | --- |
| `tutorial.zip` | 20.3 MB | `652d37fe9727` | 教程包：`Student-Server.zip`、`student-client.zip`、`vue3_springboot_beginner_tutorial (1).md` |
| `vue3_springboot_practice_project.rar` | 10.0 MB | `e8b1800c6c59` | 练习项目源码（含 `frontend/node_modules`），824 个文件 |
| `vue3_springboot_practice_project_docker.rar` | 65.6 MB | `056a0244c71a` | 带 Docker 的完整版（含 `node_modules`、`data/` 证书），1226 个文件，解压约 306 MB |
| `vue3_springboot_practice_project_docker_batch.zip` | 145 KB | `7aa797080495` | 仅源码 + Docker 配置的精简版（无 `node_modules`、无数据库数据） |

完整校验：

```bash
sha256sum archives/vue3-springboot-practice/*
```

## 注意

- `vue3_springboot_practice_project.rar` 与 `..._docker.rar` 里带 `frontend/node_modules`，这是体积大头。
- `..._docker.rar` 的 `data/` 下有本地 MySQL 用的 `ca-key.pem`、`client-key.pem`、`private_key.pem`、`server-key.pem` 等证书，是本地 Docker 开发环境自动生成的测试证书，不对应任何线上服务；如需干净版本，用 `..._docker_batch.zip`。
- 这些文件都超过仓库 README 里「单文件 >10 MB 不入库」的约定，属于按用户要求例外保留的归档。
