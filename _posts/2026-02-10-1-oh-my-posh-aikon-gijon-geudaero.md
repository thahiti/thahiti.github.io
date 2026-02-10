---
layout: post
title: "1. Oh My Posh & 아이콘 (기존 그대로)"
date: 2026-02-10 13:45:32 +0900
categories: [DiscordLog]
tags: [til, dev]
---

# 1. Oh My Posh & 아이콘 (기존 그대로)
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json" | Invoke-Expression
Import-Module Terminal-Icons

# 2. fzf 모듈 임포트 (먼저 불러와야 합니다)
Import-Module PSFzf

# 3. [수정] 키 바인딩 설정 (이 부분이 핵심입니다!)
# Ctrl+t : 파일 검색 (현재 디렉토리)
Set-PSReadLineKeyHandler -Key 'Ctrl+t' -ScriptBlock { Invoke-Fzf -Type Path }

# Ctrl+r : 히스토리 검색 (과거 명령어)
Set-PSReadLineKeyHandler -Key 'Ctrl+r' -ScriptBlock { Invoke-Fzf -Type History }

# 4. PSReadLine 설정 (기존 그대로)
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle InlineView
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
