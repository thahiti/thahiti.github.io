---
layout: post
title: "; --- 설정 시작 ---"
date: 2026-02-10 15:47:40 +0900
categories: [DiscordLog]
tags: [til, dev]
---

; --- 설정 시작 ---
#SingleInstance Force
SetTitleMatchMode, 2

; Esc 키 매핑 ($는 자기 자신을 호출하지 않게 함)
$Esc::
    ; 현재 활성화된 창의 IME 상태 확인 (1:한글, 0:영문)
    if (IME_CHECK("A"))
    {
        ; 한글이면 한/영키(vk15)를 눌러 영어로 전환
        Send, {vk15}
    }

    ; 원래 Esc 키 기능 실행 (창 닫기 등)
    ; 만약 Esc 기능은 끄고 한영전환만 하고 싶다면 아래 줄을 지우세요.
    Send, {Esc}
return

; --- IME 상태 확인 함수 (수정하지 마세요) ---
IME_CHECK(WinTitle)
{
    WinGet,hWnd,ID,%WinTitle%
    Return Send_ImeControl(ImmGetDefaultIMEWnd(hWnd),0x005,"")
}

Send_ImeControl(DefaultIMEWnd, wParam, lParam)
{
    DetectHiddenWindows, ON
    SendMessage 0x283, wParam,lParam,,ahk_id %DefaultIMEWnd%
    if (ErrorLevel = "FAIL")
        Return 0
    else
        Return ErrorLevel
}

ImmGetDefaultIMEWnd(hWnd)
{
    return DllCall("imm32\ImmGetDefaultIMEWnd", Uint,hWnd, Uint)
}
