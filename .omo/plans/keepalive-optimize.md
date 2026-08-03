# 优化：空提交 → 随机内容文件保持活跃

**状态**: 待执行  
**创建**: 2026-08-03  
**目标文件**: `.github/workflows/quark_signin.yml`

## 背景

当前"空提交保持活跃" step 用 `git commit --allow-empty` 每次产生相同内容的空提交。存在未来被 GitHub 过滤的风险。

## 方案概述

将空提交替换为：删除旧 `.github/keepalive.log` → 写入时间戳 + 随机30-50中文字符 → 正常提交。

## 改动范围

- **文件**: `.github/workflows/quark_signin.yml`
- **Step**: `空提交保持活跃` → 重写
- **影响**: 仅该 step，不涉及签到逻辑

## 具体实现

### 替换内容

**原 step**（46-54行）:

```yaml
      - name: 空提交保持活跃
        if: success() && github.event_name == 'schedule'
        run: |
          git config --local user.email "${{ github.actor_id }}+${{ github.actor }}@users.noreply.github.com"
          git config --local user.name "${{ github.actor }}"
          git remote set-url origin https://${{ github.actor }}:${{ github.token }}@github.com/${{ github.repository }}
          git pull --rebase --autostash
          git commit --allow-empty -m "CHORE: 保持运行.."
          git push
```

**新 step**:

```yaml
      - name: 保持仓库活跃（随机内容提交）
        if: success() && github.event_name == 'schedule'
        run: |
          git config --local user.email "${{ github.actor_id }}+${{ github.actor }}@users.noreply.github.com"
          git config --local user.name "${{ github.actor }}"
          git pull --rebase --autostash

          # 预置100个常用中文字符作为随机池
          CHARS="的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天四日那社义事平形相全表间样与关各重新线内数正心反你明看原又么利比或但质气第向道命此变条只没结解问意建月公无系军很情最重"

          # 删除旧 keepalive 文件（如果存在）
          if [ -f ".github/keepalive.log" ]; then
            git rm ".github/keepalive.log"
          fi

          # 生成北京时间戳
          TS=$(TZ='Asia/Shanghai' date '+%Y-%m-%d %H:%M:%S')
          # 随机取30-50个中文字符
          COUNT=$((RANDOM % 21 + 30))
          TEXT=$(echo "$CHARS" | fold -w1 | shuf -n $COUNT | tr -d '\n')

          # 写入新文件并提交
          echo "$TS | $TEXT" > ".github/keepalive.log"
          git add ".github/keepalive.log"
          git commit -m "chore: keepalive $TS"
          git push
```

### 新增文件

- `.github/keepalive.log` — 自动生成，每次签到后更新。内容示例：`2026-08-03 08:05:12 | 发年得面重加道三的定然法小要开下同部家最没`

### 顺带修复（可选，同一 step 内）

`git remote set-url origin https://...:${{ github.token }}@...` 将 token 嵌入 URL，有泄露风险。GitHub Actions 默认注入 `GITHUB_TOKEN`，`git push` 直接可用。可删掉 `git remote set-url` 行。

## 验证清单

- [ ] workflow 语法验证（YAML lint）
- [ ] 手动 `workflow_dispatch` 触发一次，确认 `.github/keepalive.log` 正确生成并推送
- [ ] 观察 GitHub 提交历史：每次提交内容不同
- [ ] 确认 60 天后 Actions 未被暂停

## 回滚方案

若方案有问题，还原为原空提交 step 即可。`.github/keepalive.log` 可删除。

## 风险

| 风险 | 等级 | 说明 |
|------|------|------|
| `shuf` 在 ubuntu-latest 不可用 | 极低 | shuf 是 coreutils 的一部分，ubuntu 标配 |
| 并发冲突 | 极低 | 每天只跑2次，间隔 > 10h |
| `.github/keepalive.log` 被误认为配置文件 | 低 | 放 `.github/` 下，命名明确 |
