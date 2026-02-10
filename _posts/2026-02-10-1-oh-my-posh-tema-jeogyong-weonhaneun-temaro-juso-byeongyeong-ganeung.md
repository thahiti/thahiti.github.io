---
layout: post
title: "1. Oh My Posh 테마 적용 (원하는 테마로 주소 변경 가능)"
date: 2026-02-10 13:33:37 +0900
categories: [DiscordLog]
tags: [til, dev]
---

# 1. Oh My Posh 테마 적용 (원하는 테마로 주소 변경 가능)
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json" | Invoke-Expression

# 2. Terminal-Icons (파일 아이콘)
Import-Module Terminal-Icons

# 3. PSReadLine (Zsh 스타일 자동완성 & 키 바인딩)
Set-PSReadLineOption -PredictionSource History   # 히스토리 기반 예측(회색 글씨)
Set-PSReadLineOption -PredictionViewStyle InlineView # 한 줄로 보기
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete # 탭 누르면 메뉴 선택

# 4. fzf (퍼지 서치) 연동
Import-Module PSFzf
Set-PSFzfOption -PSReadlineChordProvider 'Ctrl+t' # Ctrl+T로 파일 검색
Set-PSFzfOption -PSReadlineHistoryChordProvider 'Ctrl+r' # Ctrl+R로 히스토리 검색
