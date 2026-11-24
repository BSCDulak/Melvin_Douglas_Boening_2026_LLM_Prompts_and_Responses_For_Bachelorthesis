# PayPerQ | Chat Export

## Conversation Details

- **Model**: GPT-6 Luna
- **Date**: 11/23/2026, 7:50:10 AM

## Messages

### 👤 User (Message 1)

So I need some help creating a little autohotkey script for computergame automation.
While something like that already exists i am woefully unsatisfied with its performance.
When I play a game I want the script to record my mouse clicks and key presses, but the current script in particular doesn't properly check for key pressed down, key held and key released, it only sees a keypress.
This doesn't help when I want to automate movement patterns, like in factorio, I want to run to the left with a for 5 seconds but when I use the script it only registers a pressed a bunch of times and when it then runs the character instead of running for exactly 5 seconds only runs for like 1 or so second or even less because it makes (Send, {Blind} aaaaaaaaa) instead of something like a down, Sleep, 5000, a up.
It would also be cool if the script knows in what window i am when i run it, there is something called ahk_class and ahk_exe, so that when i accidently run the automation script outside of the game it doesn't just press random buttons.
structure wise it doesn't need to be that complicated, just at the top of the screen a little button for start, stop, edit.
so start button would make it start recording, stop would make it stop recording and edit would then open the file (that file could be called ~pattern.ahk, in the current script it works LogFile:=A_Temp . "\~Record.ahk" would be temporary and overwritten every time i press start again) in notpad so i can manually change some things e.g. if i did a little mistake in the movement pattern or something, and then I can save it as an .ahk file and use it for the game.


### 🤖 Assistant (Message 2)

I wrote this for **AutoHotkey v2**, which is the current version. Your current script uses v1 syntax (`Send, {Blind}`). If you only have v1 installed, you'll also need to install v2. They can be installed side by side.

It fixes your three main problems:

1. **Holding a key is recorded as one long press.** When you hold a key, Windows keeps sending repeated "key down" signals. The recorder ignores those repeats and only writes `{a down}` once. It writes `{a up}` when you actually let go, with the real time in between. Holding `a` for 5 seconds becomes `{a down}`, `Sleep 5000`, `{a up}`.
2. **It locks onto the game window.** It saves the game's `ahk_exe` and `ahk_class`, and only records input while that window is active. The generated pattern only plays inside that window and stops right away if you leave it.
3. **Simple toolbar** at the top of the screen with **Start / Stop / Edit**, plus an X button to close it.

## Recorder.ahk

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; Where the recording is written (overwritten on every recording)
global LogFile := A_ScriptDir "\~pattern.ahk"    ; or: A_Temp "\~pattern.ahk"

global Recording := false, Arming := false
global TargetHwnd := 0, TargetExe := "", TargetClass := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w60", "Start")
global BtnStop  := G.Add("Button", "x+5 w60 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w60", "Edit")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w260", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0 x" (A_ScreenWidth // 2 - 230))
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd) {
    global
    TargetHwnd  := hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    Lines := [], Held := Map(), UsedKeys := Map()
    LastTick := A_TickCount
    Arming := false, Recording := true
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    G.BackColor := "FF9999"
    StatusTxt.Value := "● REC  " TargetExe
}

StopRec() {
    global
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    } else if Recording {
        Recording := false
        IH.Stop()
        for h in MouseHotkeys
            Hotkey h, "Off"
        ; release anything still held when you stopped
        for k in Held.Clone()
            AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                       : 'Send "{Blind}{' EscKey(k) ' up}"')
        Held.Clear()
        WritePattern()
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    StatusTxt.Value := "Saved: " LogFile
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive("ahk_id " TargetHwnd))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return                                   ; was pressed before recording started
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive("ahk_id " TargetHwnd))
            return                               ; ignore clicks on the toolbar
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive("ahk_id " TargetHwnd))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)      ; no sleep before the very first event
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

WritePattern() {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    try FileDelete LogFile
    FileAppend txt, LogFile, "UTF-8"
}
```

## How to use it

1. Run `Recorder.ahk`. The toolbar appears at the top of the screen, and you can drag it by its background.
2. Go into the game and click **Start** or press **F9**. The toolbar won't take focus away from the game, so it locks onto the game straight away. The toolbar turns red and shows the game's exe.
3. Play normally, then click **Stop** or press **F9**. Clicks on the toolbar itself are never recorded.
4. Click **Edit** to open `~pattern.ahk` in Notepad. Fix anything you want, then use **Save As** to keep it under its own name. The `~pattern.ahk` file is overwritten every time you stop a recording.
5. Run the saved pattern file, go into the game and press **F8**. Press **F10** to stop it at any time.

A recorded Factorio run looks like this:

```ahk
Pattern() {
    Send "{Blind}{a down}"
    Sleep 5012
    Send "{Blind}{a up}"
    Sleep 340
    Click "812 455 Left Down"
    Sleep 95
    Click "812 455 Left Up"
}
```

## Safety: the pattern can't press keys outside the game
- **F8 only works while the game window is active.** It's tied to `ahk_exe` + `ahk_class`, so pressing it anywhere else does nothing.
- **A check runs every 50 ms during playback.** If you alt-tab or the game loses focus, the pattern stops right away and lets go of every key it was holding. You won't end up with a stuck `a` key on your desktop.
- **The recorder ignores input in other windows.** Anything you type outside the game while recording isn't saved.

## Things to know
- **Fullscreen games:** If the game uses exclusive fullscreen, the toolbar won't show on top. Use **F9** instead, or switch the game to borderless/windowed mode.
- **Stop with F9, not alt-tab:** If you alt-tab out while recording, the Alt+Tab gets recorded too. The safety check will catch it during playback, but it's cleaner not to record it.
- **If the game ignores the inputs:** Try changing `SendMode "Event"` to `SendMode "Input"` in the pattern file. If the game runs as administrator, the scripts have to run as administrator too.
- **Timing:** Each `Sleep` can be off by about 10–15 ms. That's fine for movement, but very long patterns with hundreds of steps can drift a little.
- **Mouse drags:** Only the press and release positions are saved. A drag (for example, laying belts in Factorio) plays back as a straight line between those two points.
- **Repeating:** Set `Repeat := 0` in the pattern file to loop it forever until you press F10 or leave the game.

Also check the game's rules before using it online, because some games ban automation.


### 👤 User (Message 3)

Nice, that works really well, I just tested it on my main gaming pc. I do have a travel laptop and play some games there aswell, could you modify the script in a way so that it can send me the ~pattern.ahk per email. I assume you cannot make it so that it sends the saved file, or is it possible if i save the saved files in a folder like savedpatterns? I use thunderbird on the main pc and outlook on the laptop. It would be cool if I could just have a send button and it automatically sends the file. For privacy reasons I would input my email into the script myself after you're done editing, so mark the place where I gotta put my email address with a comment EMAIL HERE before the input field.


### 🤖 Assistant (Message 4)

I added a **Save** button and a **Send** button. Saved patterns go into a `SavedPatterns` folder, and **Send** can email any of them, not just the current recording.

- **Outlook (laptop):** sends completely in the background. You don't see a window.
- **Thunderbird (main PC):** Thunderbird can't send mail by itself from the command line. The script opens a new message with the file already attached, then presses Ctrl+Enter to send it. You'll see the window flash briefly.

You only need to enter your address once, at the line marked **EMAIL HERE** near the top. The same script works on both PCs, because it uses Thunderbird when it finds it and Outlook otherwise.

## Recorder.ahk

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; e.g. "me@example.com"
global MailMethod  := "auto"              ; "auto" = Thunderbird if installed, else Outlook
                                          ; or force: "thunderbird" / "outlook"
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

global LogFile := A_ScriptDir "\~pattern.ahk"        ; current recording (overwritten each time)
global SaveDir := A_ScriptDir "\SavedPatterns"       ; your saved patterns
DirCreate SaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global TargetHwnd := 0, TargetExe := "", TargetClass := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w250", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd) {
    global
    TargetHwnd  := hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    Lines := [], Held := Map(), UsedKeys := Map()
    LastTick := A_TickCount
    Arming := false, Recording := true
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    G.BackColor := "FF9999"
    StatusTxt.Value := "● REC  " TargetExe
}

StopRec() {
    global
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    } else if Recording {
        Recording := false
        IH.Stop()
        for h in MouseHotkeys
            Hotkey h, "Off"
        ; release anything still held when you stopped
        for k in Held.Clone()
            AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                       : 'Send "{Blind}{' EscKey(k) ' up}"')
        Held.Clear()
        WritePattern()
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", (*) => SendFile(LogFile))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendFile.Bind(A_LoopFileFullPath))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")
    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

SendFile(path, *) {
    global Mailing
    if (Trim(EmailTo) = "")
        return MsgBox("Please enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!")
    if Mailing
        return
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }
    subject := "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"   ; unique -> easy to find the window
    body    := "AutoHotkey pattern " name " sent from " A_ComputerName

    method := MailMethod
    if (method = "auto")
        method := TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    ok := (method = "thunderbird") ? SendThunderbird(attach, subject, body)
                                   : SendOutlook(attach, subject, body)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", EmailTo, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    ; wait (max 30 s) for the compose window that has our unique subject in its title
    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500                                   ; give the attachment time to load
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"                              ; Thunderbird: Send Now
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body) {
    try {
        if !ProcessExist("outlook.exe") {        ; start Outlook so the mail actually leaves the outbox
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)                 ; 0 = olMailItem
        mail.To      := EmailTo
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive("ahk_id " TargetHwnd))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return                                   ; was pressed before recording started
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive("ahk_id " TargetHwnd))
            return                               ; ignore clicks on the toolbar
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive("ahk_id " TargetHwnd))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)      ; no sleep before the very first event
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

WritePattern() {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    try FileDelete LogFile
    FileAppend txt, LogFile, "UTF-8"
}
```

## What's new

| Button | What it does |
|---|---|
| **Save** | Asks for a name and copies `~pattern.ahk` into `SavedPatterns\name.ahk`. Unlike `~pattern.ahk`, these files aren't overwritten by the next recording. |
| **Send** | Shows a menu listing the current recording and every file in `SavedPatterns`. Click one and it's emailed to you. The menu also has an option to open the folder. |

The `SavedPatterns` folder is created automatically next to `Recorder.ahk`. You can also put files there yourself and they'll appear in the Send menu.

## How to use it
1. Record → Stop → **Edit**. Make your changes in Notepad and press **Ctrl+S**. **Save** copies whatever is on disk, so unsaved Notepad changes won't be included.
2. **Save** → enter a name, for example `factorio_run_left`.
3. **Send** → pick the file. You'll get an email with a subject like "AHK pattern factorio_run_left.ahk (date)".

## One-time setup

**Thunderbird (main PC):** The first time, Thunderbird will probably ask "Are you sure you want to send this message?" Tick **"Don't ask me again"** and click Send. After that, sending is fully automatic. If it still asks, the script shows you a message explaining what to do.

**Outlook (laptop):** This only works with **classic Outlook**. The newer "new Outlook" app, which comes with recent Windows 11 installs, can't be controlled by scripts at all. If you have the new one, you'll get an error message. In that case, tell me and I can add a third option that sends directly through your mail provider's server (SMTP). That works without any mail program, but it needs an app password stored in the script. Also, if Outlook ever shows a warning that "a program is trying to send e-mail", that usually means your antivirus isn't recognized as up to date.

## Things to know
- **If the email doesn't arrive or the attachment is blocked:** Some mail providers filter script files. Set `AttachAsTxt := true`, and the file is sent as `name.ahk.txt`. Just remove the `.txt` after downloading it.
- **Mouse clicks depend on the screen size.** Clicks are saved as positions inside the game window. If the laptop's resolution or window size is different from your main PC's, recorded clicks will land in the wrong place. Keyboard movement (like holding `a` for 5 seconds) works the same on both. For click patterns, record them on the machine you'll play them on, or run the game at the same window size on both.
- **Game version on the laptop:** The pattern only plays in a window with the same `ahk_exe` and `ahk_class`. If the game's exe has a different name on the laptop (for example Steam vs. a standalone version), change the `GameWin :=` line in the pattern file.


### 👤 User (Message 5)

Can you elaborate on third option the mail provider's server SMTP thing? would it be just like trying to login to an email server like i do with thunderbird or outlook and I need the password because if I use thunderbird or outlook it has the password already saved?


### 🤖 Assistant (Message 6)

Yes, you've got it right. SMTP is the "outgoing mail" part of every email account. When Thunderbird or Outlook sends a message, it connects to your provider's SMTP server (for example `smtp.gmail.com`), logs in with your saved credentials, and hands the message over. The SMTP option skips the mail program and has the script make that same connection itself. The script has no access to the password Thunderbird or Outlook saved, so you'd have to give it its own.

## How it would work

The script needs four things:

| Setting | Example (Gmail) |
|---|---|
| Server | `smtp.gmail.com` |
| Port | 587 or 465 |
| Username | your full email address |
| Password | an **app password**, not your normal password |

When you click **Send**, the script logs in, sends the mail with the attachment, and disconnects. No window opens and no mail program is needed, so it works the same on both PCs.

## Why an app password and not your normal one

For most big providers your normal password won't work anyway. With Gmail, you now authenticate with an app password (2-Step Verification required) or OAuth 2.0. Your normal Gmail password fails with a 535 error.

An app password is a separate password the provider generates for one program. App Passwords are provider-generated 16-character passwords that give a single application access without exposing your main account password. That matters here because the password would sit in plain text in the script:

- If someone gets the script, they can send mail from your account, but they can't log into the account, change your password, or read your settings.
- You can revoke the app password at any time without changing your real password.

Thunderbird and Outlook don't need this because they use a newer login method (OAuth). Modern clients (Outlook, Thunderbird 91+, Apple Mail on macOS 10.14+) trigger a Microsoft sign-in browser popup and complete OAuth automatically. Older clients or scripts need an app-specific password. A simple script can't easily use OAuth, which is why it needs the app password.

## What depends on your provider

- **Gmail:** Works well. Turn on 2-step verification, then create an app password under Google Account → Security → App passwords.
- **GMX, web.de, most smaller providers:** Usually work fine. Some need SMTP turned on in the web settings first.
- **Outlook.com / Hotmail / Microsoft 365:** Least reliable. Microsoft is shutting down simple password logins for SMTP. For Microsoft 365, in late 2026 or 2027 Basic Authentication will no longer work and all attempts to connect will fail with an error. I'm not sure how long app passwords will keep working for personal Outlook.com accounts, so I wouldn't rely on a Microsoft account for this.

## My suggestion

Make a **separate, free email account** (Gmail, GMX, or similar) used only for sending patterns, and put its app password in the script. Mail still goes to your real address. That way:

- your main account's credentials are never stored in a file
- the worst case if the script leaks is that someone can send mail from a throwaway account
- it works on both PCs whether or not Thunderbird or Outlook is installed

I can add this as a third mode (`MailMethod := "smtp"`) that uses tools already built into Windows, so you wouldn't need to install anything. The password field would be marked with a comment just like the email field. Which provider would you use for sending? Port and encryption settings differ slightly between providers.


### 👤 User (Message 7)

Yeah that sounds great, please do. I think I would use protonmail for that if possible, if it doesn't work with protonmail then i guess gmail would be ok too.


### 🤖 Assistant (Message 8)

Unfortunately ProtonMail won't work for this unless you pay, and even then it's awkward. **Gmail is the better choice.** You can still send the patterns *to* your ProtonMail address. Only the sending account needs to be Gmail.

## Why ProtonMail doesn't work here

- **Free plan:** No Bridge and no SMTP. Free accounts can only use the web and mobile apps, with a limit of 150 messages per day.
- **Paid plan with Proton Mail Bridge:** Bridge is a small program that runs on your computer, decrypts mail locally and hands it to your email app. It would have to be installed and running on both PCs, which is what we were trying to avoid.
- **Paid plan with an SMTP token:** Proton's direct SMTP access also needs a paid plan and a custom domain on top of that. Your normal `@proton.me` address isn't enough. You'd need to own a domain like `yourname.com`.

## Setting up Gmail (about 5 minutes)

1. Create a new Gmail account just for sending, e.g. `mypatterns.sender@gmail.com`.
2. In that account, go to **Google Account → Security** and turn on **2-Step Verification**.
3. Search for **"App passwords"** in the Google Account settings and create one (name it something like "Pattern script").
4. Google shows a 16-character password like `abcd efgh ijkl mnop`. Copy it **without the spaces** into the script.

## The code

This uses `CDO.Message`, which is built into Windows, so you don't need to install anything. I wrote it for **AutoHotkey v2**. If your script is v1 (for example, `MsgBox, text` without parentheses), tell me and I'll convert it.

```autohotkey
; ============================================================
;  SMTP SETTINGS – only used when MailMethod := "smtp"
; ============================================================
SmtpServer   := "smtp.gmail.com"
SmtpPort     := 465                                ; Gmail: 465 with SSL
SmtpUser     := "mypatterns.sender@gmail.com"      ; <-- CHANGE: the SENDING Gmail address
SmtpPassword := "abcdefghijklmnop"                 ; <-- CHANGE: 16-char app password, NO spaces
                                                   ;     (NOT your normal Gmail password!)

; ============================================================
;  Sends a mail with an attachment via SMTP
;  Returns true on success, false on failure
; ============================================================
SendViaSmtp(toAddress, subject, body, attachmentPath := "") {
    global SmtpServer, SmtpPort, SmtpUser, SmtpPassword
    schema := "http://schemas.microsoft.com/cdo/configuration/"

    try {
        msg := ComObject("CDO.Message")
        cfg := msg.Configuration.Fields

        settings := Map(
            "sendusing",             2,             ; 2 = send over the network
            "smtpserver",            SmtpServer,
            "smtpserverport",        SmtpPort,
            "smtpusessl",            true,
            "smtpauthenticate",      1,             ; 1 = username/password login
            "sendusername",          SmtpUser,
            "sendpassword",          SmtpPassword,
            "smtpconnectiontimeout", 30
        )
        for name, value in settings
            cfg.Item(schema name).Value := value
        cfg.Update()

        msg.From     := SmtpUser
        msg.To       := toAddress
        msg.Subject  := subject
        msg.TextBody := body
        msg.TextBodyPart.Charset := "utf-8"        ; so umlauts etc. display correctly

        if (attachmentPath != "") {
            if !FileExist(attachmentPath) {
                MsgBox("Attachment not found:`n" attachmentPath, "SMTP error", "Icon!")
                return false
            }
            msg.AddAttachment(attachmentPath)      ; needs the FULL path
        }

        msg.Send()
        return true
    } catch as e {
        MsgBox("Sending failed:`n`n" e.Message
            . "`n`nCheck: app password correct? Internet connection? "
            . "2-Step Verification enabled on the Gmail account?"
            , "SMTP error", "Icon!")
        return false
    }
}
```

## Adding it to your Send button

Add a branch for the new mode where your script checks `MailMethod`. I can't see your exact variable names from earlier, so rename these to match:

```autohotkey
if (MailMethod = "smtp") {
    if SendViaSmtp(MailTo, "Pattern: " PatternName, "Pattern attached.", PatternFile)
        ToolTip("Sent!"), SetTimer(() => ToolTip(), -2000)
}
else if (MailMethod = "outlook") {
    ; ... your existing Outlook code
}
else {
    ; ... your existing Thunderbird code
}
```

## Notes

- **Port 465 rather than 587:** The built-in Windows mail component works reliably with Gmail's SSL port 465 but not with 587.
- **Security:** Anyone who opens the script can see the password. That's why we're using a separate throwaway account. If the file ever ends up somewhere it shouldn't, delete the app password in Google and make a new one.
- **First test:** If you get an "authentication" error, it's almost always the app password. Check that the spaces are removed and that you didn't use the normal Gmail password.
- **Both PCs:** You can copy the same script to both machines. Nothing else needs to be set up on either one.

If you paste your current script, I can add this directly to it so you don't have to fit the pieces in yourself.


### 👤 User (Message 9)

this was the last version of the script please add the new things directly:

#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; e.g. "me@example.com"
global MailMethod  := "auto"              ; "auto" = Thunderbird if installed, else Outlook
                                          ; or force: "thunderbird" / "outlook"
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

global LogFile := A_ScriptDir "\~pattern.ahk"        ; current recording (overwritten each time)
global SaveDir := A_ScriptDir "\SavedPatterns"       ; your saved patterns
DirCreate SaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global TargetHwnd := 0, TargetExe := "", TargetClass := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w250", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd) {
    global
    TargetHwnd  := hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    Lines := [], Held := Map(), UsedKeys := Map()
    LastTick := A_TickCount
    Arming := false, Recording := true
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    G.BackColor := "FF9999"
    StatusTxt.Value := "● REC  " TargetExe
}

StopRec() {
    global
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    } else if Recording {
        Recording := false
        IH.Stop()
        for h in MouseHotkeys
            Hotkey h, "Off"
        ; release anything still held when you stopped
        for k in Held.Clone()
            AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                       : 'Send "{Blind}{' EscKey(k) ' up}"')
        Held.Clear()
        WritePattern()
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", (*) => SendFile(LogFile))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendFile.Bind(A_LoopFileFullPath))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")
    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

SendFile(path, *) {
    global Mailing
    if (Trim(EmailTo) = "")
        return MsgBox("Please enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!")
    if Mailing
        return
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }
    subject := "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"   ; unique -> easy to find the window
    body    := "AutoHotkey pattern " name " sent from " A_ComputerName

    method := MailMethod
    if (method = "auto")
        method := TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    ok := (method = "thunderbird") ? SendThunderbird(attach, subject, body)
                                   : SendOutlook(attach, subject, body)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", EmailTo, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    ; wait (max 30 s) for the compose window that has our unique subject in its title
    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500                                   ; give the attachment time to load
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"                              ; Thunderbird: Send Now
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body) {
    try {
        if !ProcessExist("outlook.exe") {        ; start Outlook so the mail actually leaves the outbox
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)                 ; 0 = olMailItem
        mail.To      := EmailTo
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive("ahk_id " TargetHwnd))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return                                   ; was pressed before recording started
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive("ahk_id " TargetHwnd))
            return                               ; ignore clicks on the toolbar
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive("ahk_id " TargetHwnd))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)      ; no sleep before the very first event
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

WritePattern() {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    try FileDelete LogFile
    FileAppend txt, LogFile, "UTF-8"
}


### 🤖 Assistant (Message 10)

I've added Gmail sending (SMTP) to your script below. The settings block, `SendFile()` and the new `SendSmtp()` function changed. Everything else is the same as your version.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; where the patterns go, e.g. "me@proton.me"
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 465                               ; Gmail: 465 with SSL (587 does NOT work with CDO)
global SmtpUser     := ""                                ; <-- the SENDING Gmail address, e.g. "mypatterns.sender@gmail.com"
global SmtpPassword := ""                                ; <-- 16-char app password, NO spaces (NOT your normal password!)

global LogFile := A_ScriptDir "\~pattern.ahk"        ; current recording (overwritten each time)
global SaveDir := A_ScriptDir "\SavedPatterns"       ; your saved patterns
DirCreate SaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global TargetHwnd := 0, TargetExe := "", TargetClass := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w250", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd) {
    global
    TargetHwnd  := hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    Lines := [], Held := Map(), UsedKeys := Map()
    LastTick := A_TickCount
    Arming := false, Recording := true
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    G.BackColor := "FF9999"
    StatusTxt.Value := "● REC  " TargetExe
}

StopRec() {
    global
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    } else if Recording {
        Recording := false
        IH.Stop()
        for h in MouseHotkeys
            Hotkey h, "Off"
        ; release anything still held when you stopped
        for k in Held.Clone()
            AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                       : 'Send "{Blind}{' EscKey(k) ' up}"')
        Held.Clear()
        WritePattern()
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", (*) => SendFile(LogFile))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendFile.Bind(A_LoopFileFullPath))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")
    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

SendFile(path, *) {
    global Mailing
    if (Trim(EmailTo) = "")
        return MsgBox("Please enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!")
    if Mailing
        return
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }
    subject := "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"   ; unique -> easy to find the window
    body    := "AutoHotkey pattern " name " sent from " A_ComputerName

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body)
    else
        ok := SendOutlook(attach, subject, body)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
}

; ---------------- Gmail / SMTP (built into Windows, nothing to install) ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }
    schema := "http://schemas.microsoft.com/cdo/configuration/"
    try {
        msg := ComObject("CDO.Message")
        cfg := msg.Configuration.Fields
        settings := Map(
            "sendusing",             2,                                  ; 2 = send over the network
            "smtpserver",            SmtpServer,
            "smtpserverport",        SmtpPort,
            "smtpusessl",            true,
            "smtpauthenticate",      1,                                  ; 1 = username/password login
            "sendusername",          SmtpUser,
            "sendpassword",          StrReplace(SmtpPassword, " "),     ; strips spaces just in case
            "smtpconnectiontimeout", 30
        )
        for key, value in settings
            cfg.Item[schema key] := value
        cfg.Update()

        msg.From     := SmtpUser
        msg.To       := EmailTo
        msg.Subject  := subject
        msg.TextBody := body
        msg.TextBodyPart.Charset := "utf-8"
        msg.AddAttachment(attach)                                       ; needs the FULL path
        msg.Send()                                                      ; toolbar freezes for a few seconds while sending
        return true
    } catch as e {
        MsgBox "Sending via Gmail failed:`n`n" e.Message "`n`n"
             . "Check: app password correct (no spaces, not your normal password)?`n"
             . "2-Step Verification enabled on the Gmail account?`n"
             . "Internet connection / firewall blocking port 465?", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", EmailTo, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    ; wait (max 30 s) for the compose window that has our unique subject in its title
    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500                                   ; give the attachment time to load
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"                              ; Thunderbird: Send Now
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body) {
    try {
        if !ProcessExist("outlook.exe") {        ; start Outlook so the mail actually leaves the outbox
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)                 ; 0 = olMailItem
        mail.To      := EmailTo
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive("ahk_id " TargetHwnd))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return                                   ; was pressed before recording started
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive("ahk_id " TargetHwnd))
            return                               ; ignore clicks on the toolbar
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive("ahk_id " TargetHwnd))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)      ; no sleep before the very first event
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

WritePattern() {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    try FileDelete LogFile
    FileAppend txt, LogFile, "UTF-8"
}
```

## What you need to fill in
Fill in these three lines near the top:
- **`EmailTo`:** Where the patterns should arrive. This can be your ProtonMail address.
- **`SmtpUser`:** The Gmail account that sends them.
- **`SmtpPassword`:** The 16-character app password from Google.

## What changed
- **`MailMethod` now defaults to `"smtp"`:** Patterns are sent straight through Gmail, so you don't need Thunderbird or Outlook on either PC.
- **`"auto"` now tries Gmail first:** If the Gmail details are filled in, it uses them. Otherwise it falls back to Thunderbird, then Outlook, like before.
- **New `SendSmtp()` function:** It uses the same inputs as your Thunderbird and Outlook functions, so `AttachAsTxt` and the unique subject line still work.
- **Spaces in the app password are removed automatically,** in case you paste it the way Google shows it.

## Things to know
- **The toolbar freezes while sending.** This usually lasts 1–3 seconds and is normal. Sending happens in one step, and the status text updates when it's finished.
- **Check your ProtonMail spam folder the first time.** Proton may flag mail from a brand-new Gmail account. If it lands there, mark it as "not spam" and future patterns should arrive normally.


### 👤 User (Message 11)

So when I try to use the smtp I get Sending via Gmail failed: (0x800CCE05) The requested body part was not found in this message. Source: CDO.Message.1 Check: app password correct (no spaces, not your normal password)? 2-Step Verification enabled on the Gmail account? Internet connection / firewall blocking port 465.

I have made an app password and when i check on google it says that it hasn't been used yet. I dunno about my firewall maybe blocking port 465 - how could i check that?


### 🤖 Assistant (Message 12)

This error isn't caused by your firewall or your password. The script crashed before it ever contacted Gmail, which is why Google says the app password hasn't been used yet.

## What actually went wrong

Code `0x800CCE05` ("requested body part was not found") is a local CDO error. It comes from this line in `SendSmtp()`:

```autohotkey
msg.TextBodyPart.Charset := "utf-8"
```

This line sets the encoding of the email text, and CDO sometimes can't find that text part yet when it runs. The script stopped there, before `msg.Send()`, so no connection was ever made. My error message then listed firewall and password causes that didn't apply here. That was misleading, sorry.

## The fix

Replace your whole `SendSmtp()` function with this version. It sets the encoding in a way that doesn't crash, and the error message now tells you which kind of problem happened:

```autohotkey
SendSmtp(attach, subject, body) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }
    schema := "http://schemas.microsoft.com/cdo/configuration/"
    stage  := "building the message"
    try {
        msg := ComObject("CDO.Message")
        cfg := msg.Configuration.Fields
        settings := Map(
            "sendusing",             2,                                  ; 2 = send over the network
            "smtpserver",            SmtpServer,
            "smtpserverport",        SmtpPort,
            "smtpusessl",            true,
            "smtpauthenticate",      1,                                  ; 1 = username/password login
            "sendusername",          SmtpUser,
            "sendpassword",          StrReplace(SmtpPassword, " "),     ; strips spaces just in case
            "smtpconnectiontimeout", 30
        )
        for key, value in settings
            cfg.Item[schema key] := value
        cfg.Update()

        msg.BodyPart.Charset := "utf-8"                                 ; set charset BEFORE the body (no crash)
        msg.From     := SmtpUser
        msg.To       := EmailTo
        msg.Subject  := subject
        msg.TextBody := body
        msg.AddAttachment(attach)                                       ; needs the FULL path

        stage := "connecting to Gmail"
        msg.Send()                                                      ; toolbar freezes for a few seconds while sending
        return true
    } catch as e {
        code := Format("0x{:08X}", e.Number & 0xFFFFFFFF)
        if (stage = "building the message")
            hint := "This is a script problem, not a password/firewall problem."
        else if (code = "0x80040213")
            hint := "Could not connect to smtp.gmail.com:465.`nCheck internet / firewall / antivirus (port 465)."
        else if (code = "0x80040217" || code = "0x8004020E")
            hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
        else
            hint := "Check internet, app password and 2-Step Verification."
        MsgBox "Sending via Gmail failed while " stage ":`n`n"
             . e.Message "`n`n" hint, "Recorder", "Icon!"
        return false
    }
}
```

## If you still want to check port 465

You probably won't need this now, but you can test it in about 10 seconds. Open **PowerShell** from the Start menu and run:

```powershell
Test-NetConnection smtp.gmail.com -Port 465
```

Look at the last line of the output:
- **`TcpTestSucceeded : True`**: The port is open and nothing is blocking it.
- **`TcpTestSucceeded : False`**: Something is blocking the connection. This is usually antivirus "mail shield" software (Avast, AVG, ESET), a work or school network, or a VPN.

## What to expect next

- **It works:** The status bar shows "Sent: …" and Google will show the app password as used.
- **Login error:** If you see "Gmail rejected the login," check that `SmtpUser` is the full Gmail address and that the app password was copied correctly. If in doubt, create a new app password.
- **Connection error:** If you see "Could not connect," run the PowerShell test above.


### 👤 User (Message 13)

When I try to run it now it tells me before even starting:
Warning this local variable appears to never be assigned a value.
 Specifically: TBPath 

198: method := MailMethod
199: If (method = "auto"
200: method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"
202: StatusTxt.Value := "Sending " name " via " method "..."
203: If (method = "smtp")

is that an issue or can i ignore that warning?


### 🤖 Assistant (Message 14)

Don't ignore it. The warning means AutoHotkey can't find a **function** named `TBPath()` anywhere in the script, so it treats `TBPath` as an empty variable.

Right now it won't cause a problem, because SMTP is configured and the line stops at `"smtp"` without calling `TBPath()`. But if you ever clear the SMTP settings, or SMTP is skipped for some other reason, the script will crash on that line.

## Most likely cause

When you replaced `SendSmtp()`, part of the code around it probably got deleted or broken. That usually happens in one of two ways:

1. **Too much was selected**, so the `TBPath()` function (which likely sat just above or below `SendSmtp()`) got deleted along with it.
2. **The curly braces don't match.** If an old closing `}` is missing or there's an extra one, the functions after `SendSmtp()` end up *inside* it. Code elsewhere can't see them, so you get this same warning.

## How to check

1. Press **Ctrl+F** in your editor and search for `TBPath(`.
   - **Only found on line 200:** The function was deleted. Add it back (see below).
   - **Also found as `TBPath() {` somewhere:** The function still exists, so the braces are the problem. Look at the end of your new `SendSmtp()`. It should end with `}` (closing the `catch`) and then one more `}` (closing the function), followed by the next function. Remove any leftover lines or extra `}` from the old version.
2. Search the same way for the other send functions (for example `SendThunderbird` / `SendOutlook`, or whatever they're called in your script). If those are also missing, the paste removed more than one function, and it's easiest to restore from a backup or redo the paste carefully.

## If `TBPath()` is missing, add this back

Paste it as its own function, **outside** any other function. The very end of the script is fine:

```autohotkey
TBPath() {
    ; Returns the path to thunderbird.exe, or "" if Thunderbird isn't installed
    candidates := [
        A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe",
        EnvGet("ProgramFiles(x86)") "\Mozilla Thunderbird\thunderbird.exe"
    ]
    for p in candidates
        if FileExist(p)
            return p
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    return ""
}
```

It checks the two usual install folders first, then looks in the Windows registry. If Thunderbird isn't installed, it returns an empty string, so "auto" mode falls through to Outlook.

Once it's fixed, the warning should disappear when you start the script. If you get a similar warning for another name, it's the same problem (a missing function or mismatched braces). Paste the warning here and I'll help you sort it out.


### 👤 User (Message 15)

oh yeah, i overwrote it accidently, I have restored to an older version and properly copy pasted now. Now when I try to start it I get the following:
Error: This value of type "Error" has no property names "Number"

254: Catch as e
254 {
255: code := Format("0x{:08X}", e.Number & 0xFFFFFFFF)
256:If (stage = "building the message")
257: hint := "This is a script problem, not a password/firewall problem."


### 🤖 Assistant (Message 16)

This one is my mistake. The error handler I gave you has a bug, and that bug is hiding the real error.

## What's happening

1. Something inside the `try` block failed. We don't know what yet.
2. The script jumped to the `catch` block to show my error message.
3. My `catch` code assumed every error has a `Number` property (an error code). Only COM/Windows errors have one. This one is a plain AutoHotkey `Error`, so the handler crashed trying to read `e.Number`, and the original error message was lost.

A plain `Error` (rather than a COM/OS error) usually means a script-side problem, such as a variable that's missing or empty, or a line AutoHotkey couldn't handle. It doesn't mean Gmail rejected anything.

## The fix

Replace the whole `catch as e { ... }` block at the end of `SendSmtp()` with this version. It only reads `Number` when the error actually has one, and it shows the details we need to find the real cause:

```autohotkey
    } catch as e {
        code := e.HasProp("Number") ? Format("0x{:08X}", e.Number & 0xFFFFFFFF) : "(none)"
        if (stage = "building the message")
            hint := "This is a script problem, not a password/firewall problem."
        else if (code = "0x80040213")
            hint := "Could not connect to smtp.gmail.com:465.`nCheck internet / firewall / antivirus (port 465)."
        else if (code = "0x80040217" || code = "0x8004020E")
            hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
        else
            hint := "Check internet, app password and 2-Step Verification."
        MsgBox "Sending via Gmail failed while " stage ":`n`n"
             . "Error type: " Type(e) "`n"
             . "Message: "    e.Message "`n"
             . "Code: "       code "`n"
             . "Where: "      e.What "`n"
             . "Extra: "      e.Extra "`n"
             . "Line: "       e.Line "`n`n"
             . hint, "Recorder", "Icon!"
        return false
    }
}
```

The final `}` closes the function. Make sure you replace the old closing braces rather than adding extra ones, or you'll get the brace problem from before.

## Next step

Run it again and try sending. You'll get a message box that shows the **real** error with its line number. Paste that message here, along with the line it points to, and I can tell you exactly what's wrong.

If it says something like "variable has not been assigned a value" and names `SmtpServer`, `SmtpPort` or `EmailTo`, check that those are still defined in your SETTINGS section. Restoring the older version could have changed them.


### 👤 User (Message 17)

ok, now i am getting when i try to send:
Sending via Gmail failed while building the message:

Error type: Error
Message: (0x800CCE05)
The requested body part was not found in this message.

Source: CDO.Message.1
Code: (none)
Where:
Extra: Charset
Line: 244

This is a script problem, not a password/firewall problem.


### 🤖 Assistant (Message 18)

The new error box shows the real cause: it's still the charset line. **Extra: Charset** and **Line 244** point to `msg.BodyPart.Charset := "utf-8"`. My earlier "fix" moved the problem without solving it. CDO won't accept a charset before the message has any content, which gives the same "body part not found" error. Sorry about that.

## The fix

In `SendSmtp()`, **delete** this line:

```autohotkey
        msg.BodyPart.Charset := "utf-8"                                 ; set charset BEFORE the body (no crash)
```

Then set the charset **after** the body text exists, and wrap it in `try` so it can never block sending again. That part of the function should look like this:

```autohotkey
        msg.From     := SmtpUser
        msg.To       := EmailTo
        msg.Subject  := subject
        msg.TextBody := body
        try msg.TextBodyPart.Charset := "utf-8"                         ; optional: only matters for ä/ö/ü/é etc.
        msg.AddAttachment(attach)                                       ; needs the FULL path
```

The charset only matters if your subject or body contains special characters like umlauts or accents. If setting it ever fails, the `try` skips it and the email still goes out.

## A second small fix for the error box

Your box showed **Code: (none)**. AutoHotkey reports COM errors as a plain `Error` and puts the code in the message text, like `(0x800CCE05)`, not in a `Number` property. So the "login rejected" and "could not connect" hints would never trigger. Replace the first line of the `catch` block:

```autohotkey
        code := e.HasProp("Number") ? Format("0x{:08X}", e.Number & 0xFFFFFFFF) : "(none)"
```

with this:

```autohotkey
        code := e.HasProp("Number") ? Format("0x{:08X}", e.Number & 0xFFFFFFFF)
              : RegExMatch(e.Message, "i)0x[0-9A-F]{8}", &m) ? StrUpper(m[0]) : "(none)"
```

This reads the code out of the message text when there's no `Number`, so the correct hint shows up.

## What to expect now

Try sending again. The script should now get past "building the message" and actually contact Gmail.

- **Success:** You'll see "Sent: …", and Google will show the app password as used.
- **"Failed while connecting to Gmail":** The message box will now point to the actual cause (login or port 465). Paste it here if it's not clear what to do.


### 👤 User (Message 19)

So, I have done the changes and still get an error message: to clear things up I will send the full script and the error message i get when i try to send so you can take a look at it.
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := "redacted"                  ; where the patterns go, e.g. "me@proton.me"
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 465                               ; Gmail: 465 with SSL (587 does NOT work with CDO)
global SmtpUser     := "redacted"                                ; <-- the SENDING Gmail address, e.g. "mypatterns.sender@gmail.com"
global SmtpPassword := "redacted"                                ; <-- 16-char app password, NO spaces (NOT your normal password!)

global LogFile := A_ScriptDir "\~pattern.ahk"        ; current recording (overwritten each time)
global SaveDir := A_ScriptDir "\SavedPatterns"       ; your saved patterns
DirCreate SaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global TargetHwnd := 0, TargetExe := "", TargetClass := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w250", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd) {
    global
    TargetHwnd  := hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    Lines := [], Held := Map(), UsedKeys := Map()
    LastTick := A_TickCount
    Arming := false, Recording := true
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    G.BackColor := "FF9999"
    StatusTxt.Value := "● REC  " TargetExe
}

StopRec() {
    global
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    } else if Recording {
        Recording := false
        IH.Stop()
        for h in MouseHotkeys
            Hotkey h, "Off"
        ; release anything still held when you stopped
        for k in Held.Clone()
            AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                       : 'Send "{Blind}{' EscKey(k) ' up}"')
        Held.Clear()
        WritePattern()
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", (*) => SendFile(LogFile))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendFile.Bind(A_LoopFileFullPath))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")
    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

SendFile(path, *) {
    global Mailing
    if (Trim(EmailTo) = "")
        return MsgBox("Please enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!")
    if Mailing
        return
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }
    subject := "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"   ; unique -> easy to find the window
    body    := "AutoHotkey pattern " name " sent from " A_ComputerName

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body)
    else
        ok := SendOutlook(attach, subject, body)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
}

; ---------------- Gmail / SMTP (built into Windows, nothing to install) ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }
    schema := "http://schemas.microsoft.com/cdo/configuration/"
    stage  := "building the message"
    try {
        msg := ComObject("CDO.Message")
        cfg := msg.Configuration.Fields
        settings := Map(
            "sendusing",             2,                                  ; 2 = send over the network
            "smtpserver",            SmtpServer,
            "smtpserverport",        SmtpPort,
            "smtpusessl",            true,
            "smtpauthenticate",      1,                                  ; 1 = username/password login
            "sendusername",          SmtpUser,
            "sendpassword",          StrReplace(SmtpPassword, " "),     ; strips spaces just in case
            "smtpconnectiontimeout", 30
        )
        for key, value in settings
            cfg.Item[schema key] := value
        cfg.Update()

        msg.From     := SmtpUser
        msg.To       := EmailTo
        msg.Subject  := subject
        msg.TextBody := body
		try msg.BodyPart.Charset := "utf-8"
        msg.AddAttachment(attach)                                       ; needs the FULL path

        stage := "connecting to Gmail"
        msg.Send()                                                      ; toolbar freezes for a few seconds while sending
        return true
    } catch as e {
        code := e.HasProp("Number") ? Format("0x{:08X}", e.Number & 0xFFFFFFFF)
              : RegExMatch(e.Message, "i)0x[0-9A-F]{8}", &m) ? StrUpper(m[0]) : "(none)"
        if (stage = "building the message")
            hint := "This is a script problem, not a password/firewall problem."
        else if (code = "0x80040213")
            hint := "Could not connect to smtp.gmail.com:465.`nCheck internet / firewall / antivirus (port 465)."
        else if (code = "0x80040217" || code = "0x8004020E")
            hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
        else
            hint := "Check internet, app password and 2-Step Verification."
        MsgBox "Sending via Gmail failed while " stage ":`n`n"
             . "Error type: " Type(e) "`n"
             . "Message: "    e.Message "`n"
             . "Code: "       code "`n"
             . "Where: "      e.What "`n"
             . "Extra: "      e.Extra "`n"
             . "Line: "       e.Line "`n`n"
             . hint, "Recorder", "Icon!"
        return false
    }
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", EmailTo, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    ; wait (max 30 s) for the compose window that has our unique subject in its title
    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500                                   ; give the attachment time to load
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"                              ; Thunderbird: Send Now
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body) {
    try {
        if !ProcessExist("outlook.exe") {        ; start Outlook so the mail actually leaves the outbox
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)                 ; 0 = olMailItem
        mail.To      := EmailTo
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive("ahk_id " TargetHwnd))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return                                   ; was pressed before recording started
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive("ahk_id " TargetHwnd))
            return                               ; ignore clicks on the toolbar
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive("ahk_id " TargetHwnd))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)      ; no sleep before the very first event
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

WritePattern() {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    try FileDelete LogFile
    FileAppend txt, LogFile, "UTF-8"
}



**[Image]**

*(local asset — not exportable)*

*Created: 11/18/2026, 6:54:11 PM*


### 🤖 Assistant (Message 20)

Thanks for sending the full script. The screenshot shows something new: the error now says **Extra: From** instead of **Extra: Charset**. So the charset was never the real problem. CDO fails on whatever property the script sets **first** on the message. Earlier that was the charset. Now that the charset is set later, it's `msg.From`.

My guess is that the order is the cause. Your script gets `msg.Configuration` and changes its settings **before** it sets From, To, Subject and Body. Older AutoHotkey CDO scripts that are known to work do it the other way around. There's also a small v2 issue: `cfg.Item[...] := value` is the v1 style. In v2 it's safer to set the field's `.Value` directly.

## The fix

In `SendSmtp()`, replace everything from `try {` up to (but **not** including) `} catch as e {` with this:

```autohotkey
    try {
        msg := ComObject("CDO.Message")

        ; 1) Fill in the message FIRST
        msg.From     := SmtpUser
        msg.To       := EmailTo
        msg.Subject  := subject
        msg.TextBody := body

        ; 2) THEN the Gmail connection settings
        cfg := msg.Configuration.Fields
        settings := Map(
            "sendusing",             2,                                  ; 2 = send over the network
            "smtpserver",            SmtpServer,
            "smtpserverport",        SmtpPort,
            "smtpusessl",            true,
            "smtpauthenticate",      1,                                  ; 1 = username/password login
            "sendusername",          SmtpUser,
            "sendpassword",          StrReplace(SmtpPassword, " "),     ; strips spaces just in case
            "smtpconnectiontimeout", 30
        )
        for key, value in settings
            cfg.Item[schema key].Value := value                          ; v2: set .Value explicitly
        cfg.Update()

        ; 3) Attachment, then send
        msg.AddAttachment(attach)                                       ; needs the FULL path

        stage := "connecting to Gmail"
        msg.Send()                                                      ; toolbar freezes for a few seconds while sending
        return true
```

I removed the charset line completely. Your subject and body are plain text like "AHK pattern xyz.ahk (2026-…)", so it isn't needed. Leave your `catch` block as it is. It worked well, and it's what showed us where the real problem was.

## Also check the address format

CDO is fussy about the From address. In the SETTINGS section, `SmtpUser` and `EmailTo` should each be **only** the bare address, for example:

```autohotkey
global SmtpUser := "mypatterns.sender@gmail.com"
```

Don't add a name, angle brackets like `<…>`, spaces, or a trailing semicolon.

## If it still fails

Try sending again and look at the message box:

- **It says "while connecting to Gmail":** That's progress. The message was built, and the hint will tell you whether it's a login or port problem.
- **It still says "building the message":** Then CDO itself is the problem on your system. It's old Windows technology and can be unreliable on Windows 10/11. In that case, I'd suggest replacing `SendSmtp()` with a version that sends through PowerShell, which is built into Windows and handles Gmail more reliably. Let me know and I'll write it for you.


### 👤 User (Message 21)

now I get the same error message with Source CDO.Message.1 but instead of Line 244 it is Line 231.


### 🤖 Assistant (Message 22)

I think CDO is a dead end on your PC, so I'd stop trying to repair it.

## Why I'm giving up on CDO

Line 231 is now `msg.From := SmtpUser`, which is the first thing the script sets on the message. So across three attempts, CDO has failed on whatever line came first: the charset, then From in the old order, then From again in the new order. Changing the order wasn't the fix. CDO can't build a message on your system at all. It's very old Windows technology and is often broken on Windows 10/11.

I should have suggested a different approach sooner instead of moving lines around. Sorry for the back and forth.

## The new approach: send through PowerShell

PowerShell is built into Windows, so there's nothing to install. The script writes a small PowerShell file, runs it hidden, and reads back the result.

### Step 1: Change the port in SETTINGS

```autohotkey
global SmtpPort     := 587                               ; Gmail: 587 (STARTTLS) for the PowerShell method
```

This method only works with 587. Port 465 won't work.

### Step 2: Replace the whole `SendSmtp()` function

Replace everything from `SendSmtp(attach, subject, body) {` down to its last closing `}` (just above `; ---------------- Thunderbird ----------------`) with this:

```autohotkey
SendSmtp(attach, subject, body) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ; PowerShell script: reads everything from environment variables (no quoting problems)
    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $env:REC_TO
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", SmtpServer,
        "REC_PORT",   SmtpPort,
        "REC_USER",   Trim(SmtpUser),
        "REC_PASS",   StrReplace(SmtpPassword, " "),
        "REC_TO",     Trim(EmailTo),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach smtp.gmail.com:" SmtpPort ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    MsgBox "Sending via Gmail failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}
```

## How it works

- **Environment variables:** The address, password, subject and file path go to PowerShell as environment variables, not command-line text. That way spaces, quotes or special characters in file names can't break anything.
- **Password cleanup:** The password is removed from the environment right after sending, and the temporary `.ps1` file is deleted.
- **Real error messages:** If something fails, you'll see Gmail's actual reply. For example, "5.7.8 Username and Password not accepted" means a login problem.
- **Deprecation note:** Microsoft marks `Send-MailMessage` as outdated, but it still works with Gmail and an app password. The warning about this is hidden.

## If it fails, check the port

This time the error will be a real network or login message, not a script problem. If it says it couldn't connect, test the port in PowerShell:

```powershell
Test-NetConnection smtp.gmail.com -Port 587
```

`TcpTestSucceeded : True` means the port is open. In that case, the problem is the login details.


### 👤 User (Message 23)

Nice, it works now. So now I have another idea. I wanna do some statistic on my factorio button presses, so I want in the script a little statistics part at the end where it says how often I pressed a certain key. I think we can disregard mouseclicks for the statistic because they can be associated with too many different things, but hotkeys would be pretty interesting. Additionally to that I want it to automatically start when I enter factorio and stop and send the pattern (which should be saved with a different name than the normal pattern so i don't overwrite when I want to do a specific pattern) when I exit factorio, I presume the best way would be to check whether I am in the factorio window or not through the class, I think we could put which class I want automated in the config so i can use it for other games too. I am sure I will otherwise forget to start and stop the thing. And maybe a config for another reciever email for the automatic patterns so my normal email doesn't get overloaded.


### 🤖 Assistant (Message 24)

I've rewritten the whole script below. Several functions changed, and with your earlier copy-paste trouble, one full replacement is safer than many small edits.

**Before you paste:** copy your real `EmailTo`, `SmtpUser` and `SmtpPassword` somewhere, because the new version has placeholders. The SMTP port is already set to 587.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; where your normal patterns go, e.g. "me@proton.me"
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 587                               ; Gmail: 587 (STARTTLS) for the PowerShell method
global SmtpUser     := ""                                ; <-- the SENDING Gmail address
global SmtpPassword := ""                                ; <-- 16-char app password, NO spaces

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoWin     := "ahk_exe factorio.exe"             ; which game to watch (see notes: ahk_exe or ahk_class)
global AutoEmailTo := ""                                 ; receiver for auto sessions ("" = use EmailTo)

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoSkip := false
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w250", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second whether the game is open

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, winTitle := "", auto := false) {
    global
    TargetHwnd  := hwnd
    TargetWin   := winTitle != "" ? winTitle : "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    Arming := false, Recording := true, AutoSession := auto
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := auto ? "FFCC66" : "FF9999"
    StatusTxt.Value := (auto ? "● AUTO REC  " : "● REC  ") TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, AutoSkip
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    AutoSkip := true                             ; don't auto-restart until the game is closed & reopened
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendTo(file, AutoRecipient())
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global AutoSkip
    if (!AutoEnabled || Trim(AutoWin) = "")
        return
    gameOpen := WinExist(AutoWin)
    if AutoSession {
        if (Recording && !gameOpen)
            StopRec()                            ; game closed -> save + send
        return
    }
    if !gameOpen {
        AutoSkip := false                        ; game closed -> auto-start again next time
        return
    }
    if (AutoSkip || Recording || Arming || Mailing)
        return
    if (hwnd := WinActive(AutoWin))
        BeginRecording(hwnd, AutoWin, true)
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" game "_session_" FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, EmailTo))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, EmailTo))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, AutoRecipient()))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

SendTo(path, to, *) {
    global Mailing
    if (Trim(to) = "")
        return MsgBox("Please enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!")
    if Mailing
        return
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }
    subject := "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"   ; unique -> easy to find the window
    body    := "AutoHotkey pattern " name " sent from " A_ComputerName

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else
        ok := SendOutlook(attach, subject, body, to)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
}

; ---------------- Gmail / SMTP via PowerShell ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body, to) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $env:REC_TO
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", SmtpServer,
        "REC_PORT",   SmtpPort,
        "REC_USER",   Trim(SmtpUser),
        "REC_PASS",   StrReplace(SmtpPassword, " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach smtp.gmail.com:" SmtpPort ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    MsgBox "Sending via Gmail failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := to
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## What's new

### Key statistics
Every pattern file now ends with a statistics block, written as comments so the pattern still runs. It contains:
- **Duration and pace:** how long the session lasted, total key presses, and presses per minute.
- **Keys:** each key with its count and share of all presses, most used first. Holding a key down counts as one press.
- **Combinations:** key combos like `Ctrl+C` or `Shift+1`, listed separately.

Mouse clicks are still recorded in the pattern itself but aren't counted in the statistics. F9 isn't counted either.

### Automatic sessions
- **Start:** when the game window becomes active and nothing else is recording, recording starts by itself. The toolbar turns **orange** with "● AUTO REC".
- **While running:** alt-tabbing away doesn't stop it. Only input inside the game is recorded.
- **Stop:** when the game **closes**, the session is saved to `AutoPatterns\factorio_session_2026-11-18_20.15.30.ahk` and emailed to `AutoEmailTo`. Leave `AutoEmailTo` empty to use your normal `EmailTo` instead. Each session gets its own file name, so nothing is ever overwritten.
- **Pressing F9 during an auto session:** this ends the session normally (saved and sent). Auto recording then pauses until you close and restart the game, so you can record a specific manual pattern in between. Manual recordings still go to `~pattern.ahk` as before.
- **Closing the recorder mid-session:** the session file is still saved, just not sent.
- **Resending:** if a send fails (for example, no internet), use **Send → Auto sessions (resend)**.

## About the game setting

You asked about matching by window class. I set it up to match by the program file (`ahk_exe factorio.exe`) instead, because many games use generic class names that other programs share. `AutoWin` accepts either form, though, so for other games you can use:

```autohotkey
global AutoWin := "ahk_exe otherGame.exe"      ; by program file (recommended)
global AutoWin := "ahk_class SomeClassName"    ; by window class
```

To find the correct names, right-click the green AutoHotkey tray icon, choose **Window Spy**, then click into the game. It shows both `ahk_exe` and `ahk_class`.

## Good to know

- **Chat is recorded too.** Anything you type in the game window, including in-game chat or console commands, ends up in the file and the email.
- **File size:** long sessions make big files, but they should stay far below Gmail's 25 MB attachment limit. As a rough estimate, a few hours of play is a few MB.
- **The toolbar freezes briefly** while the session is being sent after the game closes. That's normal.


### 👤 User (Message 25)

Works perfectly. Now I am playing modded factorio sometimes and I gotta revert to an old version to play with some not currently updated mods. I noticed that the factorio window has a name with a version number, can we make it so that I use that window name to determine when to stop and start autorecording instead of factorio.exe so i can stop start based on whether i play with group a modded or group b unmodded, so that would need to be part of the message for the email then too and it would mean i need a few triggers to start stop automatically so it gets all the sessions i want it to get.


### 🤖 Assistant (Message 26)

Below is the full updated script with **multiple auto triggers** (I call them "profiles"). Each profile matches the window title, so the same `factorio.exe` can be told apart by version. The profile name goes into the file name, the email subject and the statistics. The statistics are also in the email text now, so you can read them without opening the attachment.

**Before you paste:** copy your real `EmailTo`, `AutoEmailTo`, `SmtpUser` and `SmtpPassword` somewhere, because the new version has placeholders again.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; where your normal patterns go, e.g. "me@proton.me"
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 587                               ; Gmail: 587 (STARTTLS) for the PowerShell method
global SmtpUser     := ""                                ; <-- the SENDING Gmail address
global SmtpPassword := ""                                ; <-- 16-char app password, NO spaces

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w280", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
    StatusTxt.Value := AutoSession ? "● AUTO REC [" Opt(profile, "Name") "]  " TargetExe
                                   : "● REC  " TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendAuto(file)
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, EmailTo))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, EmailTo))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; normal patterns
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName)
}

; auto sessions: profile in subject, statistics in the mail text
SendAuto(path, *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, RecipientFor(pname), "[" pname "] Session " rest, BodyFromFile(path))
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

SendMail(path, to, subject, body) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nEnter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else
        ok := SendOutlook(attach, subject, body, to)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
    return ok
}

; ---------------- Gmail / SMTP via PowerShell ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body, to) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $env:REC_TO
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", SmtpServer,
        "REC_PORT",   SmtpPort,
        "REC_USER",   Trim(SmtpUser),
        "REC_PASS",   StrReplace(SmtpPassword, " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach smtp.gmail.com:" SmtpPort ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    MsgBox "Sending via Gmail failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := to
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## Setting up your profiles

**1. Check the real window titles.** I'm not 100% sure how Factorio writes its title for each version, so check before you trust the default settings:
1. Start the old (modded) version.
2. Right-click the AutoHotkey tray icon and choose **Window Spy**, then click into the game.
3. Note the title shown at the top, for example `Factorio 1.1.110`.
4. Do the same for the current version.

**2. Adjust the `Title` lines if needed.** `Title` only has to match part of the window title. The default settings mean:

| Profile | `Title` | Matches |
|---|---|---|
| Modded | `^Factorio.* 1\.1\.` | title starts with "Factorio" and contains " 1.1." |
| Vanilla | `^Factorio.* 2\.0\.` | the same, but with " 2.0." (also matches "Factorio: Space Age 2.0.x") |
| Factorio | *(empty)* | any other Factorio version (catch-all) |

Write a dot as `\.`, because a plain `.` means "any character" in these patterns. Capital or small letters don't matter.

**3. Give each profile its own receiver (optional).** Set `EmailTo` inside a profile, for example `EmailTo: "modded@..."`. Profiles left at `""` use `AutoEmailTo`, and if that is empty too, your normal `EmailTo`.

**For other games,** add another line, for example:

```autohotkey
    {Name: "Satisfactory", Exe: "FactoryGame-Win64-Shipping.exe", Title: "", EmailTo: ""}
```

Every line except the last needs a comma at the end, and the order matters: **the first matching profile wins.** That's why the catch-all has to stay at the bottom.

## What changed

- **File names:** sessions are saved as `AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk`, so the files are grouped by profile in the folder.
- **Email subject:** for example `[Modded] Session factorio 2026-11-18 20.15.30`. You can set up a Gmail filter on `[Modded]` or `[Vanilla]` to sort them into labels.
- **Email text:** contains the profile, the exact window title (so the exact version), start time, duration and the full key statistics. You don't need to open the attachment to see them.
- **Stopping:** an auto session ends when *that specific game window* closes.
- **Pressing F9 during an auto session:** only that game window is skipped. If you close it and start the other version, auto recording starts again right away.
- **Resending:** works as before through **Send → Auto sessions (resend)**, and goes to the right receiver for that profile. Your older `factorio_session_...` files still work and go to `AutoEmailTo`.

## Good to know

- **The version is checked once, when recording starts.** If Factorio briefly showed a title without the version while loading, the session could end up under the catch-all `Factorio` profile. If you see that happening, tell me, or simply delete the catch-all line.
- **Tables in the email:** the columns may not line up in Gmail because it uses a proportional font. In the `.ahk` file they line up properly.


### 👤 User (Message 27)

Works perfectly. Another thing, I already have a ton of autohotkey scripts running and they are starting to clog up the "show hidden icons" are on the bottom right of my taskbar. can you make it so that this script doesn't shop up there seeing as it is already closeable by the x button on on top in its own menu?


### 🤖 Assistant (Message 28)

Yes, you only need to add one line. Put `#NoTrayIcon` at the very top of the script, right under `#Requires`:

```autohotkey
#Requires AutoHotkey v2.0
#NoTrayIcon
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window
```

Leave the rest of the script as it is. After you reload it, the recorder won't add an icon to the hidden icons area. You can still close it with the **X** button on the toolbar.

## Optional: a backup way to close it

Without a tray icon, the X button is the only way to close the recorder. If the toolbar ever gets dragged off-screen, you'd have to use Task Manager. To avoid that, you can add a hotkey to close it, for example **Ctrl+Shift+F9**. Put this line next to the existing `F9::` line:

```autohotkey
^+F9::ExitApp()                      ; Ctrl+Shift+F9 = close the recorder
```

Closing it this way still saves a running session, the same as clicking X.

## Good to know

- **Window Spy:** earlier I told you to open it by right-clicking the tray icon, which won't be there anymore. Instead, type **Window Spy** into the Windows Start menu, or open **AutoHotkey Dash** from the Start menu and choose Window Spy there.
- **Your other scripts:** you can add `#NoTrayIcon` to any other script that you don't need to pause, reload or exit from the tray, or that has its own way to close. This is better than Windows' own taskbar settings, because Windows treats every AutoHotkey script as the same program, so it can only hide or show all of their icons together.
- **Recorded pattern files:** these are separate scripts and still show their own tray icon while they run. I left that alone because they're meant to run only briefly.


### 👤 User (Message 29)

Nice, works perfectly. Now I am a very private person and I am getting a little jittery when it comes to sending game names, window names and so on through email. is there a way we could have options to rename things to something else like in the config for the global AutoProfiles?


### 🤖 Assistant (Message 30)

Yes. The safest way is to leave your real files alone and add an **"Export shareable copy"** feature. You pick the files you want to send, and the script writes copies to a `Shareable` folder with your real names swapped for the fake names you choose. Your originals keep the real window and exe names, so they still work. You only ever email the copies.

I put the name list in its own `privacy.ini` file rather than in the global AutoProfiles config. That config contains your real names, so it's one of the files that needs cleaning before you send it. If the name list lived inside it, you could end up emailing the list itself.

## 1. Add the hotkey

Put this next to your existing `F9::` line:

```autohotkey
^+F10::ExportShareable()             ; Ctrl+Shift+F10 = make privacy-safe copies
```

## 2. Add the code at the bottom of the script

```autohotkey
; ===================== PRIVACY / ALIASES =====================
PrivacyIni := A_ScriptDir "\privacy.ini"

LoadAliases() {
    if !FileExist(PrivacyIni) {
        FileOpen(PrivacyIni, "w", "UTF-16").Write("
        (
[Aliases]
; Format:   real text = replacement
; Matching ignores upper/lower case. Longer entries are applied first.
; Add the exe name AND the window title separately, e.g.:
; MySecretGame.exe = Game1.exe
; My Secret Game - Chapter 2 = Game 1
; My Secret Game = Game 1

[Options]
HideUserName=1
HideComputerName=1
)")
        MsgBox "Created privacy.ini - add your aliases there, then export again.", "Privacy", "Iconi"
        Run 'notepad.exe "' PrivacyIni '"'
        return ""
    }

    aliases := []
    try section := IniRead(PrivacyIni, "Aliases")
    catch
        section := ""
    Loop Parse section, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";")
            continue
        if !(pos := InStr(line, "="))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        fake := Trim(SubStr(line, pos + 1))
        if (real != "")
            aliases.Push([real, fake])
    }

    if IniRead(PrivacyIni, "Options", "HideUserName", 1)
        aliases.Push(["\Users\" A_UserName "\", "\Users\User\"])
    if IniRead(PrivacyIni, "Options", "HideComputerName", 1)
        aliases.Push([A_ComputerName, "PC"])

    ; Sort longest first so "My Game 2" is replaced before "My Game"
    Loop aliases.Length - 1 {
        i := A_Index
        Loop aliases.Length - i {
            j := A_Index
            if StrLen(aliases[j][1]) < StrLen(aliases[j + 1][1]) {
                tmp := aliases[j], aliases[j] := aliases[j + 1], aliases[j + 1] := tmp
            }
        }
    }
    return aliases
}

Anonymize(text, aliases) {
    for pair in aliases
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

ExportShareable() {
    aliases := LoadAliases()
    if (aliases = "")
        return

    files := FileSelect("M3", A_ScriptDir, "Pick files to make privacy-safe copies of")
    if !files.Length
        return

    outDir := A_ScriptDir "\Shareable"
    DirCreate outDir
    count := 0
    for f in files {
        SplitPath f, &name
        text := FileRead(f, "UTF-8")
        FileOpen(outDir "\" Anonymize(name, aliases), "w", "UTF-8").Write(Anonymize(text, aliases))
        count++
    }
    MsgBox count " cleaned copies saved to:`n" outDir, "Privacy", "Iconi"
    Run outDir
}
```

## How to use it

1. Press **Ctrl+Shift+F10** once. This creates `privacy.ini` and opens it in Notepad.
2. Add a line for each thing you want hidden, like `RealName = FakeName`. Be sure to include:
   - the **exe name** (`ahk_exe`)
   - the **window title**, plus any longer variations of it
   - any **profile names** in your AutoProfiles config that give away the game
3. Save the file, then press **Ctrl+Shift+F10** again. Pick the files you want to send, for example the recorded pattern files and the AutoProfiles config.
4. The `Shareable` folder opens. Send the files from there. File names get cleaned too.

## Good to know

- **Never send `privacy.ini`**, since it lists every real name next to its fake one.
- **Shareable copies won't run on your PC** because the window names in them are fake. That's expected, since they're only for sending. Keep using your originals.
- **Your Windows username and computer name** are hidden automatically. The username is only replaced inside paths like `C:\Users\YourName\`, so a short name like "Al" won't break other words. If your name shows up somewhere else, add it as a normal alias.
- **Check the copies before you send them** the first time. Open one in Notepad and use Ctrl+F to search for the real game name. If anything is left, add an alias for that exact text.

If you'd prefer a toolbar button instead of the hotkey, paste the part of the script that builds the toolbar and I'll add one.


### 👤 User (Message 31)

can we instead make it part of the whole script so i just have to edit the script itself, I want it all to be in one .ahk?
like putting the aliases for window name and exe at the global AutoProfiles? I am not really concerned about computer names since those are already arbitrary anyways. to be clear i am not gonna share the script itself, i just don't want emails showing what games (window name and exe) i play when it sends per email.


### 🤖 Assistant (Message 32)

Yes, that's simpler. Everything stays in the one `.ahk` file, and the fake names are only applied to the email right before it's sent. The script keeps using the real window and exe names to find your games, so nothing else changes.

## 1. Remove the previous privacy code

Delete these from the script:
- the `^+F10::ExportShareable()` hotkey line
- the whole `; ===== PRIVACY / ALIASES =====` section at the bottom

You can also delete `privacy.ini` and the `Shareable` folder if they were created.

## 2. Add the alias list next to your global AutoProfiles config

```autohotkey
; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Upper/lower case doesn't matter. Add the exe AND the window title.
; Lines starting with ; are ignored.
EmailAliases := "
(
MySecretGame.exe = Game1.exe
My Secret Game = Game 1
AnotherGame.exe = Game2.exe
Another Game - Launcher = Game 2
)"
EmailAliasList := ParseAliases(EmailAliases)
```

Put one alias on each line. You don't need quotes or commas. Swap in your real names on the left side.

## 3. Add these two functions at the bottom of the script

```autohotkey
; ===================== EMAIL PRIVACY =====================
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; Longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}
```

## 4. Clean the email right before it's sent

Find the function in your script that sends the email. Searching for `CDO.Message` or `Subject` should get you there. At the start of that function, run the subject and body through `Anonymize()`. For example, if the function looks like `SendEmail(subject, body)`, add:

```autohotkey
    subject := Anonymize(subject)
    body    := Anonymize(body)
```

Use whatever variable names your function actually uses. Because every email goes through that one function, this covers all of them, no matter which part of the script triggers it.

## Good to know

- **Include every version of the name.** Window titles sometimes change, for example "My Game - Chapter 2" or "My Game (DX12)". Add any longer versions as separate lines. Because longer entries are replaced first, they won't get cut in half.
- **Attachments aren't cleaned.** If an email attaches a log file or a screenshot, the fake names only apply to the subject and body. A screenshot would still show the game itself.
- **Send yourself a test email** after setting it up, and check that the real names don't appear anywhere in it.

If you can't find the email function, or it builds the message in more than one place, paste it here and I'll add the lines for you.


### 👤 User (Message 33)

Here is the full script again so you can put in the new parts for anonymization:

#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; where your normal patterns go, e.g. "me@proton.me"
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 587                               ; Gmail: 587 (STARTTLS) for the PowerShell method
global SmtpUser     := ""                                ; <-- the SENDING Gmail address
global SmtpPassword := ""                                ; <-- 16-char app password, NO spaces

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w280", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
    StatusTxt.Value := AutoSession ? "● AUTO REC [" Opt(profile, "Name") "]  " TargetExe
                                   : "● REC  " TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendAuto(file)
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, EmailTo))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, EmailTo))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; normal patterns
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName)
}

; auto sessions: profile in subject, statistics in the mail text
SendAuto(path, *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, RecipientFor(pname), "[" pname "] Session " rest, BodyFromFile(path))
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

SendMail(path, to, subject, body) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nEnter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name
    attach := path
    if AttachAsTxt {
        attach := A_Temp "\" name ".txt"
        FileCopy path, attach, 1
    }

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else
        ok := SendOutlook(attach, subject, body, to)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    Mailing := false
    return ok
}

; ---------------- Gmail / SMTP via PowerShell ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body, to) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $env:REC_TO
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", SmtpServer,
        "REC_PORT",   SmtpPort,
        "REC_USER",   Trim(SmtpUser),
        "REC_PASS",   StrReplace(SmtpPassword, " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach smtp.gmail.com:" SmtpPort ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    MsgBox "Sending via Gmail failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := to
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}


### 🤖 Assistant (Message 34)

Below is the full script with anonymization added. Everything is still in the one `.ahk`, and the alias list is in SETTINGS right below `AutoProfiles`.

**Why the attachment is changed too:** each pattern file has the game's exe name, window class and window title written inside it, in the `GameWin :=` line and the Statistics block. Auto-session file names also include the exe name, for example `Modded_factorio_...ahk`. Fixing only the subject and body would leave the real names in the attachment. So the script now sends a cleaned temporary copy of the file. The original files in `AutoPatterns` and `SavedPatterns` are never changed.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; where your normal patterns go, e.g. "me@proton.me"
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 587                               ; Gmail: 587 (STARTTLS) for the PowerShell method
global SmtpUser     := ""                                ; <-- the SENDING Gmail address
global SmtpPassword := ""                                ; <-- 16-char app password, NO spaces

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----                       ; NEW
; Only affects what is SENT (subject, text, attachment + its file name).
; Your local files and the game detection above keep the real names.
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Tip: the game's base name alone (e.g. "factorio") also catches "factorio.exe",
;      the window title and the file name in one go.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AnonymizeAttachment := true   ; true  = also clean the text INSIDE the attached .ahk
                                     ;         (its GameWin line then shows the fake name, so that emailed
                                     ;          copy won't run in the game unless you change that line back)
                                     ; false = only subject, email text and attachment FILE NAME are cleaned

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; NEW: cleaned copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global BtnSend  := G.Add("Button", "x+5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w280", "Idle   (F9 = Start/Stop)")
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
; drag the toolbar by clicking on its background
OnMessage(0x201, (wp, lp, msg, hwnd) => (hwnd = G.Hwnd ? PostMessage(0xA1, 2, , , G) : ""))

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
    StatusTxt.Value := AutoSession ? "● AUTO REC [" Opt(profile, "Name") "]  " TargetExe
                                   : "● REC  " TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendAuto(file)
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return
    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, EmailTo))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, EmailTo))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; normal patterns
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName)
}

; auto sessions: profile in subject, statistics in the mail text
SendAuto(path, *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, RecipientFor(pname), "[" pname "] Session " rest, BodyFromFile(path))
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

SendMail(path, to, subject, body) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nEnter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name

    ; ---- NEW: privacy - everything that leaves the PC goes through Anonymize() ----
    subject := Anonymize(subject)
    body    := Anonymize(body)
    attach  := PrepareAttachment(path, name)
    if !attach {
        Mailing := false
        StatusTxt.Value := "Sending failed: " name
        return false
    }

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."   ; toolbar only (local) -> real name is fine
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else
        ok := SendOutlook(attach, subject, body, to)
    StatusTxt.Value := ok ? "Sent: " name : "Sending failed: " name
    if ok
        try FileDelete attach                    ; remove the cleaned temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY (NEW) =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a cleaned temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(Anonymize(name), '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if AnonymizeAttachment
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- Gmail / SMTP via PowerShell ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body, to) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $env:REC_TO
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", SmtpServer,
        "REC_PORT",   SmtpPort,
        "REC_USER",   Trim(SmtpUser),
        "REC_PASS",   StrReplace(SmtpPassword, " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach smtp.gmail.com:" SmtpPort ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    MsgBox "Sending via Gmail failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := to
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## What changed
- **SETTINGS:** added the `EmailAliases` list, `AnonymizeAttachment` and `MailTempDir`.
- **`SendMail()`:** the subject and email text now go through `Anonymize()`. The attachment is a cleaned temp copy made by `PrepareAttachment()`, which cleans both the file name and the text inside it. All three ways of sending use `SendMail()`, so they're all covered.
- **New section `EMAIL PRIVACY`** with three functions: `ParseAliases()`, `Anonymize()` and `PrepareAttachment()`.
- Nothing else changed. Game detection, recording and your local files still use the real names.

## Setting up the aliases
For your setup, one line is probably enough. Remove the `;` in front of it:
```
factorio = Game1
```
Because upper and lower case are ignored, this one line changes `factorio.exe` to `Game1.exe`, `Factorio 2.0.x` to `Game1 2.0.x`, and `Modded_factorio_...ahk` to `Modded_Game1_...ahk`.

Things to check:
- **Window class:** the `GameWin` line contains `ahk_class ...`. Open any pattern file and look at that line. If the class name gives away the game, add it as another alias line.
- **Profile names** like "Modded" or "Factorio" show up in the subject. A profile named "Factorio" would be caught by the alias above. Anything else, rename it in `AutoProfiles` or add an alias for it.
- **Avoid very short aliases.** An alias like `rust = Game2` would also change "t**rust**" anywhere in the email. Use names long enough that they only match the game.
- **The emailed pattern won't run as-is** while `AnonymizeAttachment := true`, because its `GameWin` line now has the fake name. Use the local copy in `AutoPatterns`/`SavedPatterns`, or set the option to `false` if you'd rather keep the file contents real and only hide the subject, text and file name.

Send yourself one test email and search it for the real names. Check the subject, the text, the attachment name and the text inside the attachment.


### 👤 User (Message 35)

Can we add a little input box left to the send button to overwrite receiver when i press send so I don't have to edit the script when I wanna share a pattern?


### 🤖 Assistant (Message 36)

I added a receiver box to the toolbar, left of **Send**. If you type an address, everything you send from the Send menu goes there. If you leave it empty, your normal receivers from SETTINGS are used.

**Why I changed how the toolbar takes focus:** the toolbar is set to never take focus, so clicks don't pull you out of the game. That would also stop you typing in the box. So when you click the box, the toolbar takes focus. When you click back into the game, it goes back to never taking focus.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; where your normal patterns go, e.g. "me@proton.me"
                                          ; (several receivers: separate with , or ;)
global MailMethod  := "smtp"              ; "smtp"        = send directly via Gmail (no mail program needed)
                                          ; "auto"        = smtp if set up below, else Thunderbird, else Outlook
                                          ; "thunderbird" / "outlook" = force that program
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- Gmail SMTP (only used for "smtp" / "auto") ----
global SmtpServer   := "smtp.gmail.com"
global SmtpPort     := 587                               ; Gmail: 587 (STARTTLS) for the PowerShell method
global SmtpUser     := ""                                ; <-- the SENDING Gmail address
global SmtpPassword := ""                                ; <-- 16-char app password, NO spaces

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Only affects what is SENT (subject, text, attachment + its file name).
; Your local files and the game detection above keep the real names.
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Tip: the game's base name alone (e.g. "factorio") also catches "factorio.exe",
;      the window title and the file name in one go.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AnonymizeAttachment := true   ; true  = also clean the text INSIDE the attached .ahk
                                     ;         (its GameWin line then shows the fake name, so that emailed
                                     ;          copy won't run in the game unless you change that line back)
                                     ; false = only subject, email text and attachment FILE NAME are cleaned

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; cleaned copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""
global PrevActive := 0                                   ; NEW: window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; NEW: receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w280", "Idle   (F9 = Start/Stop)")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; NEW: grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; CHANGED: drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; NEW: give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; NEW: while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
    StatusTxt.Value := AutoSession ? "● AUTO REC [" Opt(profile, "Name") "]  " TargetExe
                                   : "● REC  " TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendAuto(file)                               ; always the normal receivers (box is ignored here)
}

; ---------------- Toolbar focus handling (NEW) ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return

    ; CHANGED: receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    header := override != "" ? "To: " override : "To: default receivers (SETTINGS)"
    m.Add(header, (*) => 0), m.Disable(header)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; normal patterns
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName)
}

; auto sessions: profile in subject, statistics in the mail text
; CHANGED: optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname), "[" pname "] Session " rest, BodyFromFile(path))
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

SendMail(path, to, subject, body) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nType one into the box next to Send, or enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name

    ; privacy - everything that leaves the PC goes through Anonymize()
    subject := Anonymize(subject)
    body    := Anonymize(body)
    attach  := PrepareAttachment(path, name)
    if !attach {
        Mailing := false
        StatusTxt.Value := "Sending failed: " name
        return false
    }

    method := MailMethod
    if (method = "auto")
        method := SmtpConfigured() ? "smtp" : TBPath() ? "thunderbird" : "outlook"

    StatusTxt.Value := "Sending " name " via " method "..."   ; toolbar only (local) -> real name is fine
    if (method = "smtp")
        ok := SendSmtp(attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else
        ok := SendOutlook(attach, subject, body, to)
    StatusTxt.Value := ok ? "Sent: " name " -> " to : "Sending failed: " name
    if ok
        try FileDelete attach                    ; remove the cleaned temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a cleaned temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(Anonymize(name), '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if AnonymizeAttachment
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- Gmail / SMTP via PowerShell ----------------
SmtpConfigured() => (Trim(SmtpUser) != "" && Trim(SmtpPassword) != "")

SendSmtp(attach, subject, body, to) {
    if !SmtpConfigured() {
        MsgBox "SMTP is not set up yet.`nFill in SmtpUser and SmtpPassword in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ; CHANGED: To is split into a list -> several receivers possible
    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", SmtpServer,
        "REC_PORT",   SmtpPort,
        "REC_USER",   Trim(SmtpUser),
        "REC_PASS",   StrReplace(SmtpPassword, " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "Gmail rejected the login.`nCheck SmtpUser (full Gmail address) and the 16-char app password."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach smtp.gmail.com:" SmtpPort ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    MsgBox "Sending via Gmail failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(to, "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := RegExReplace(to, "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## How to use it
1. Click the box. It shows grey placeholder text when empty.
2. Type an address, like `friend@mail.com`. For several people, separate them with `,` or `;`.
3. Press **Enter** to go back to the game, or just click **Send**.
4. The Send menu now starts with a grey line showing who will get the email, like `To: friend@mail.com`. Check it before you pick a file.
5. To go back to your normal receivers, click the box and press **Esc**. That clears the box and switches back to the game.

## What the box applies to
- **Applies:** anything sent through the Send menu, including the "Auto sessions (resend)" submenu.
- **Doesn't apply:** sessions that are sent automatically when the game closes. Those always go to the receivers in SETTINGS, so you can't accidentally send your own sessions to a friend.
- The box keeps its text until you clear it. After a restart of the recorder it starts empty.
- If an address doesn't look valid (for example, it's missing the `@`), you get a warning and nothing is sent.

## Other changes
- `EmailTo`, `AutoEmailTo` and the per-profile `EmailTo` in SETTINGS can now also hold several addresses.
- The status bar now shows who the email went to, like `Sent: x.ahk -> friend@mail.com`. Only you see the toolbar.
- The aliases don't change email addresses. They only change the subject, the email text and the attachment.


### 👤 User (Message 37)

Ok, so 2 rather embarrassing things happened. 
1. The manual pattern email contained my replacement patterns so people were confused why it said game1 instead of factorio.
2. the manual pattern email was sent from my automatic email instead of the regular email i use.

how can we fix those 2 issues?


### 🤖 Assistant (Message 38)

Both problems had the same cause: the script treated every email the same way.

1. **"Game1" instead of "factorio":** the privacy aliases were applied to every email. They're meant for your automatic sessions, but shared patterns need the real game name. Your friends also need the real name inside the file, because the `GameWin` line is how the pattern finds the game window.
2. **Wrong sender:** there was only one sending account (`SmtpUser`). Gmail always sends from the account you log in with, so every email came from your automatic address.

Now there are two sending accounts and two alias switches, one for each kind of email:

| | Sent from | Aliases |
|---|---|---|
| **Shared patterns** (Send menu) | `ShareAccount`, your regular email | off: real names |
| **Automatic sessions** (and resending them) | `SessionAccount`, your automatic email | on |

If `ShareAccount` isn't filled in, the script shows an error and sends nothing. It never quietly falls back to the other account. The Send menu also shows **From:** and **To:** at the top, so you can check both before anything goes out.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""
global PrevActive := 0                                   ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w380", "Idle   (F9 = Start/Stop)")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
    StatusTxt.Value := AutoSession ? "● AUTO REC [" Opt(profile, "Name") "]  " TargetExe
                                   : "● REC  " TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendAuto(file)                               ; always the normal receivers (box is ignored here)
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; NEW: show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(no receiver set)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    autoFrom := "From: " AccountLabel(SessionAccount)
    autoMenu.Add(autoFrom, (*) => 0), autoMenu.Disable(autoFrom)
    autoMenu.Add()
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: session account, aliases, statistics in the mail text
; optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname)
        , "[" pname "] Session " rest, BodyFromFile(path)
        , SessionAccount, AliasesOnSessions)
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus / status: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp")
        return Trim(Opt(acc, "User")) != "" ? Trim(Opt(acc, "User")) : "(account not set up!)"
    return method " (its default account)"
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nType one into the box next to Send, or enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        StatusTxt.Value := "Sending failed: " name
        return false
    }

    method := Opt(acc, "Method")
    from   := AccountLabel(acc)
    StatusTxt.Value := "Sending " name " from " from "..."
    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    StatusTxt.Value := ok ? "Sent: " name "  " from " -> " to : "Sending failed: " name
    try FileDelete attach                        ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(to, "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := RegExReplace(to, "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## What you need to fill in
The old `SmtpServer`, `SmtpPort`, `SmtpUser` and `SmtpPassword` settings are gone. Instead:

1. **`SessionAccount`:** copy your old `SmtpUser` and `SmtpPassword` here. This is your automatic address.
2. **`ShareAccount`:** put your regular address here, plus an app password **made in that account**. Each Gmail account has its own app passwords, and it needs 2-Step Verification turned on. Your automatic account's password won't work for it.
3. If your regular email isn't Gmail, change `Server` and `Port` to that provider's SMTP settings. Some providers (Proton, for example) only allow SMTP on paid plans or through their own helper app. In that case, use `Method: "thunderbird"` or `"outlook"`, and the email is sent from that program's default account.

## How it works now
- **Shared patterns** go out with the real game name and a working `GameWin` line, so they run for your friends without any edits.
- **Automatic sessions** still use the aliases and your automatic address. That includes resending one from the "Auto sessions (resend)" submenu, which shows its own **From:** line at the top.
- If you ever want aliases on shared patterns too, set `AliasesOnShared := true`.

Emails that already went out can't be changed. If anyone is still confused, you could resend those patterns now. They'll come from your regular address with the real names.


### 👤 User (Message 39)

It is very good that ShareAccount doesn't fall back to the other account but think I remember we had a auto setting so that when smtp doesn't work it tries to default to thunderbird first and then to outlook, can we do that for the share account. I only made a smtp for the automatic


### 🤖 Assistant (Message 40)

I've set up `ShareAccount` so it can use `Method: "auto"`. It tries SMTP first, then Thunderbird, then Outlook. Because you didn't set up SMTP for the share account, it skips that step and goes straight to Thunderbird.

**It still never uses `SessionAccount`.** The fallback only switches between ways of sending from your regular address. Before using Thunderbird or Outlook, the script checks that the program actually has an account for that address. If it doesn't, the script skips that program, so a shared pattern can't go out from your automatic address.

## Change 1: the two account settings
Replace the `ShareAccount` and `SessionAccount` blocks in SETTINGS:

```autohotkey
; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with User/Password below
;             "thunderbird" / "outlook" = use only that program
;             "auto"        = try smtp (only if User+Password are filled in) -> Thunderbird -> Outlook
;   From:     the address the mail MUST come from (needed for Thunderbird/Outlook)
;             Thunderbird/Outlook are skipped if they don't have an account with this address
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address for smtp ("" = no smtp for this account)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "auto",
    From:     "",                  ; <- your REGULAR email address
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",                  ; empty = skip smtp, go straight to Thunderbird
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    From:     "",                  ; may stay empty for smtp (User is used)
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
```

## Change 2: `AccountLabel` and `SendMail`
Replace both functions. They sit directly below `BodyFromFile`:

```autohotkey
; which account is this (for messages)
AccName(acc) => (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"

; the address this account sends from
AccFrom(acc) => Trim(Opt(acc, "From")) != "" ? Trim(Opt(acc, "From")) : Trim(Opt(acc, "User"))

; the order in which the sending methods are tried
MethodChain(acc) {
    m := StrLower(Trim(Opt(acc, "Method")))
    if (m != "auto")
        return [m]
    chain := []
    if SmtpConfigured(acc)
        chain.Push("smtp")
    chain.Push("thunderbird", "outlook")
    return chain
}

ChainText(acc) {
    s := ""
    for m in MethodChain(acc)
        s .= (s = "" ? "" : " -> ") m
    return s
}

; short text for menus / status: who the mail comes from
AccountLabel(acc) {
    addr := AccFrom(acc)
    return (addr != "" ? addr : "(no address set!)") "   [" ChainText(acc) "]"
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nType one into the box next to Send, or enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false

    which := AccName(acc)
    from  := AccFrom(acc)
    chain := MethodChain(acc)
    for m in chain
        if !(m ~= "^(smtp|thunderbird|outlook)$") {
            MsgBox 'Unknown Method "' Opt(acc, "Method") '" for ' which '.`nUse "auto", "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
            return false
        }
    if (from = "") {
        MsgBox which " has no address set - nothing was sent.`n"
             . "Fill in From (your address) for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }

    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        StatusTxt.Value := "Sending failed: " name
        return false
    }

    ; try each method of THIS account in order (never the other account)
    ; result: 1 = sent, 0 = failed -> try next, -1 = handed to you (don't try next -> no double mail)
    result := 0, errors := "", used := ""
    for m in chain {
        StatusTxt.Value := "Sending " name " from " from " via " m "..."
        err := ""
        if (m = "smtp")
            result := SendSmtp(acc, attach, subject, body, to, &err)
        else if (m = "thunderbird")
            result := SendThunderbird(attach, subject, body, to, from, &err)
        else
            result := SendOutlook(attach, subject, body, to, from, &err)
        used := m
        if (result != 0)
            break
        errors .= "- " m ": " err "`n`n"
    }

    if (result = 1) {
        StatusTxt.Value := "Sent via " used ": " name "  " from " -> " to
    } else if (result = -1) {
        StatusTxt.Value := "Waiting for you in " used ": " name
        MsgBox err, "Recorder", "Iconi"
    } else {
        StatusTxt.Value := "Sending failed: " name
        MsgBox "Nothing was sent from " from ".`n`nTried:`n" errors, "Recorder", "Icon!"
    }
    if (result != -1)                            ; Thunderbird may still need the file
        try FileDelete attach
    Mailing := false
    return result = 1
}
```

## Change 3: SMTP, Thunderbird and Outlook
Replace everything from `; ---------------- SMTP via PowerShell ----------------` down to the end of `SendOutlook`, just before `; ---------------- Event handlers`:

```autohotkey
; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to, &err) {
    which := AccName(acc)
    if !SmtpConfigured(acc) {
        err := "User / Password of " which " are not filled in."
        return 0
    }
    if !FileExist(attach) {
        err := "Attachment not found: " attach
        return 0
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return 1

    e := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(e, "5.7.8") || InStr(e, "not accepted") || InStr(e, "authenticat"))
        hint := "The server rejected the login - check User (full address) and the app password OF THAT account."
    else if (InStr(e, "connect") || InStr(e, "timed out") || InStr(e, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") " - check internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification."
    err := e "`n  " hint
    return 0
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; does any Thunderbird profile have an account (identity) with this address?
TBHasIdentity(addr) {
    Loop Files A_AppData "\Thunderbird\Profiles\*", "D" {
        prefs := A_LoopFileFullPath "\prefs.js"
        if !FileExist(prefs)
            continue
        try txt := FileRead(prefs, "UTF-8")
        catch
            continue
        if RegExMatch(txt, 'i)"mail\.identity\.[^"]+\.useremail",\s*"\Q' addr '\E"')
            return true
    }
    return false
}

SendThunderbird(attach, subject, body, to, from, &err) {
    exe := TBPath()
    if !exe {
        err := "Thunderbird is not installed."
        return 0
    }
    if !TBHasIdentity(from) {
        err := "Thunderbird has no account for " from " (it would send from another address, so it was skipped)."
        return 0
    }
    to := RegExReplace(to, "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("from='{}',to='{}',subject='{}',body='{}',attachment='{}'", from, to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        err := "Thunderbird's compose window did not appear."
        return 0
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        err := "Thunderbird's compose window is open but couldn't be focused.`nCheck that From shows " from " and press Send yourself."
        return -1
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return 1
    err := "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic."
    return -1
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to, from, &err) {
    try RegRead("HKCR\Outlook.Application\CLSID")
    catch {
        err := "Classic Outlook is not installed."
        return 0
    }
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol := ComObject("Outlook.Application")
        account := 0
        for a in ol.Session.Accounts
            if (StrLower(Trim(a.SmtpAddress)) = StrLower(from)) {
                account := a
                break
            }
        if !account {
            err := "Outlook has no account for " from " (it would send from another address, so it was skipped)."
            return 0
        }
        mail := ol.CreateItem(0)
        mail.SendUsingAccount := account             ; force the right sender
        mail.To      := RegExReplace(to, "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return 1
    } catch as e {
        err := e.Message " (only classic Outlook can be automated, not the 'new Outlook' app)"
        return 0
    }
}
```

## What you need to do
- In `ShareAccount`, put your regular address in **`From`**. Leave `User` and `Password` empty.
- Your regular account needs to be set up in Thunderbird. If Thunderbird has more than one account, the script picks the one that matches `From`.
- The Send menu now shows the order it will try, for example: **From: you@… [thunderbird -> outlook]**.

## How the fallback works
- **Moves on to the next method** if a method can't be used: SMTP isn't filled in or fails, Thunderbird isn't installed or doesn't have your address, or the compose window never opens.
- **Stops without trying the next method** once Thunderbird's compose window is open but it can't press Send itself. For example, Thunderbird might ask for confirmation. Otherwise Outlook would send the same email again. You'll get a message telling you to finish in Thunderbird.
- **If everything fails**, one message lists why each method didn't work, and nothing is sent.

## Test before you send to friends
The script picks the Thunderbird account using Thunderbird's `from=` option. I'm fairly sure that option works, but I haven't been able to test it. The Thunderbird check also only looks in the default profile folder (`%APPDATA%\Thunderbird\Profiles`). Put your own address in the receiver box and send one pattern to yourself. If it arrives from your regular address, everything is working. If it says Thunderbird has no account for your address even though it does, tell me and I'll adjust the check.


### 👤 User (Message 41)

I am a little bit confused why do I need to put my regular email in the from part of the share account, it used to just go to my thunderbird by itself when i did manual send.


### 🤖 Assistant (Message 42)

You're right, it worked before without it. In your setup you don't actually need it. Here's why I added it, and a change that makes it optional again.

## Why I added `From`
Before, the manual send only used Thunderbird. Thunderbird used its **default account**, which is your regular email, so that was fine.

With `"auto"`, the script can now also fall back to **Outlook**. I can't tell which account Outlook uses by default. If your automatic address were set up in Outlook, a shared pattern could go out from it, and you told me that must never happen. Putting the address in `From` let the script check that the account matches before sending. But a check like that doesn't need you to type your address in.

## The fix: leave `From` empty and use the default account
If `From` is empty:
- **Thunderbird** works like before and uses its default account. The script first reads which address that is. If it turns out to be your **automatic** address, it refuses to send.
- **Outlook** is only used if it has exactly **one** account and that account isn't your automatic address. Otherwise it's skipped, because it might be the wrong account.
- If you ever do fill in `From`, it works like the version I gave you last time.

### Change 1: in SETTINGS
```autohotkey
global ShareAccount := {
    Method:   "auto",
    From:     "",                  ; empty = use Thunderbird's default account (like before)
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
```

### Change 2: in `AccountLabel`
Replace the line `return (addr != "" ? addr : "(no address set!)") ...` with:
```autohotkey
    return (addr != "" ? addr : "default account") "   [" ChainText(acc) "]"
```

### Change 3: in `SendMail`
**Delete** this whole block:
```autohotkey
    if (from = "") {
        MsgBox which " has no address set - nothing was sent.`n"
             . "Fill in From (your address) for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
```
Put this in its place:
```autohotkey
    ; the address a shared pattern must NEVER come from
    avoid := (acc = ShareAccount) ? StrLower(AccFrom(SessionAccount)) : ""
    shown := (from != "") ? from : "default account"
```
Then, in the sending loop, change these lines:
```autohotkey
        StatusTxt.Value := "Sending " name " from " shown " via " m "..."
        ...
        else if (m = "thunderbird")
            result := SendThunderbird(attach, subject, body, to, from, avoid, &err)
        else
            result := SendOutlook(attach, subject, body, to, from, avoid, &err)
```
Change `from` to `shown` in the "Sent via" status and in the "Nothing was sent from" message.

### Change 4: Thunderbird
Add this new function above `SendThunderbird`:
```autohotkey
; the address of Thunderbird's default account ("" = couldn't find out)
TBDefaultAddress() {
    ini := A_AppData "\Thunderbird\profiles.ini"
    if !FileExist(ini)
        return ""
    txt := FileRead(ini, "UTF-8")
    rel := ""
    if RegExMatch(txt, "m)^\[Install[^\]]*\]\R(?:(?!\[).*\R)*?Default=(.+)$", &m)
        rel := Trim(m[1])
    else if RegExMatch(txt, "m)^Path=(.+)\R(?:(?!\[).*\R)*?Default=1", &m)
        rel := Trim(m[1])
    if (rel = "")
        return ""
    rel := StrReplace(rel, "/", "\")
    dir := InStr(rel, ":") ? rel : A_AppData "\Thunderbird\" rel
    if !FileExist(dir "\prefs.js")
        return ""
    p := FileRead(dir "\prefs.js", "UTF-8")
    if !RegExMatch(p, 'mail\.accountmanager\.defaultaccount",\s*"([^"]+)"', &a)
        return ""
    if !RegExMatch(p, 'mail\.account\.\Q' a[1] '\E\.identities",\s*"([^",]+)', &i)
        return ""
    if !RegExMatch(p, 'mail\.identity\.\Q' i[1] '\E\.useremail",\s*"([^"]+)"', &e)
        return ""
    return e[1]
}
```
In `SendThunderbird`, change the first line and replace the `TBHasIdentity` check:
```autohotkey
SendThunderbird(attach, subject, body, to, from, avoid, &err) {
    exe := TBPath()
    if !exe {
        err := "Thunderbird is not installed."
        return 0
    }
    if (from = "") {
        from := TBDefaultAddress()
        if (from = "") {
            err := "Couldn't find out which account is Thunderbird's default, so it was skipped."
            return 0
        }
    } else if !TBHasIdentity(from) {
        err := "Thunderbird has no account for " from " (it would send from another address, so it was skipped)."
        return 0
    }
    if (avoid != "" && StrLower(from) = avoid) {
        err := "Thunderbird's account is your AUTOMATIC address (" from ") - skipped so shared patterns never come from it."
        return 0
    }
    ; ... rest stays the same
```

### Change 5: Outlook
Change the first line of `SendOutlook` to:
```autohotkey
SendOutlook(attach, subject, body, to, from, avoid, &err) {
```
Then replace the `account := 0 ... if !account {...}` part with:
```autohotkey
        account := 0
        if (from = "") {
            if (ol.Session.Accounts.Count = 1)
                account := ol.Session.Accounts.Item(1)
            else {
                err := "Outlook has several accounts and From is empty - skipped (can't tell which one is right)."
                return 0
            }
        } else {
            for a in ol.Session.Accounts
                if (StrLower(Trim(a.SmtpAddress)) = StrLower(from)) {
                    account := a
                    break
                }
            if !account {
                err := "Outlook has no account for " from " (it would send from another address, so it was skipped)."
                return 0
            }
        }
        if (avoid != "" && StrLower(Trim(account.SmtpAddress)) = avoid) {
            err := "Outlook's account is your AUTOMATIC address - skipped so shared patterns never come from it."
            return 0
        }
```

## What changes for you
Nothing to fill in. The Send menu will show **default account [thunderbird -> outlook]**, and it goes to Thunderbird like before.

**One limit:** The "never from the automatic address" check only works if the automatic address is filled in as `User` in `SessionAccount`. It should already be there, because you use SMTP for it. I haven't been able to test the part that reads Thunderbird's default account. If Thunderbird gets skipped with "Couldn't find out which account is Thunderbird's default", just put your address in `From` and it'll work.


### 👤 User (Message 43)

can you do all those changes for me?


### 🤖 Assistant (Message 44)

Yes, I can. The problem is that I can't see your full script right now. I only have the pieces I showed in my last message. Parts like the rest of `SendMail`, `SendThunderbird` and `SendOutlook` aren't in front of me, and if I guessed at them I could break something that works today.

**Paste your whole current script here** and I'll send it back with all five changes made, ready to copy over your old file.

### Before you paste
- **Remove your passwords.** Replace anything in the `Password:` fields with `"xxx"`. You don't need to share them, and I don't need them for this. After you get the script back, put your real passwords in again.
- **If the script is too long for one message,** split it into two or three parts. Tell me "part 1 of 2" and so on, and I'll wait until I have all of it.

When I send it back, I'll also tell you exactly which lines I changed, so you can see that nothing else was touched.


### 👤 User (Message 45)

#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""
global PrevActive := 0                                   ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
global StatusTxt := G.Add("Text", "x+10 yp+5 w380", "Idle   (F9 = Start/Stop)")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    StatusTxt.Value := "Waiting for game window... click into it"
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
    StatusTxt.Value := AutoSession ? "● AUTO REC [" Opt(profile, "Name") "]  " TargetExe
                                   : "● REC  " TargetExe
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
        StatusTxt.Value := "Idle   (F9 = Start/Stop)"
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        StatusTxt.Value := "Recorded -> ~pattern.ahk"
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SplitPath file, &name
    StatusTxt.Value := "Saved session: " name
    SendAuto(file)                               ; always the normal receivers (box is ignored here)
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
    StatusTxt.Value := "Saved: " name
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; NEW: show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(no receiver set)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    autoFrom := "From: " AccountLabel(SessionAccount)
    autoMenu.Add(autoFrom, (*) => 0), autoMenu.Disable(autoFrom)
    autoMenu.Add()
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: session account, aliases, statistics in the mail text
; optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname)
        , "[" pname "] Session " rest, BodyFromFile(path)
        , SessionAccount, AliasesOnSessions)
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus / status: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp")
        return Trim(Opt(acc, "User")) != "" ? Trim(Opt(acc, "User")) : "(account not set up!)"
    return method " (its default account)"
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nType one into the box next to Send, or enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        StatusTxt.Value := "Sending failed: " name
        return false
    }

    method := Opt(acc, "Method")
    from   := AccountLabel(acc)
    StatusTxt.Value := "Sending " name " from " from "..."
    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    StatusTxt.Value := ok ? "Sent: " name "  " from " -> " to : "Sending failed: " name
    try FileDelete attach                        ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(to, "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := RegExReplace(to, "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}


### 🤖 Assistant (Message 46)

I've got the full script now, and your passwords are empty, so nothing private was shared. Thanks for that.

**But I can't see the list of five changes.** Our earlier messages, including the one where I listed them, aren't visible to me now. I only have your script and my last reply. I don't want to guess which five changes you mean, because if I guess wrong I could change parts that already work.

One clue: the script already has a line marked `; NEW: show sender AND receiver before anything is sent` in `SendMenu()`. So you may have already added some of the changes.

### What I need from you (pick one)
- **Paste my earlier message that lists the changes.** Scroll up, copy it, and paste it here. This is the easiest option.
- **Or describe what you want in your own words.** For example: "the toolbar should …" or "auto sessions should …" A short list is enough.

Once I have that, I'll:
1. Send back the **complete script** with every change made, ready to copy over your old file
2. List **exactly which functions and lines changed**, so you can see nothing else was touched
3. Point out any change that's **already in your script**, so it doesn't get added twice

Your script is already saved in this chat, so you don't need to paste it again.


### 👤 User (Message 47)

You know what, I give up - let's just trim this thing down to size so we don't have a circle conversation and then I will try to fix the thing with the ShareAccount and whatnot.
Actually we can also trip the gui while we are at it, that'll be a double win.
when we send there is text on the right saying Sent: blah blah and sending method and sending and whatnot all this information is not needed. let's just cut that part off the gui.
let's do that for now and then see if it is short enough again for you to put in the other things


### 🤖 Assistant (Message 48)

I removed the status text on the right of the toolbar and every line that updated it. Nothing else changed, so recording, saving and sending work the same as before. The toolbar is now narrower and ends at the **X** button.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""
global PrevActive := 0                                   ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SendAuto(file)                               ; always the normal receivers (box is ignored here)
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; NEW: show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(no receiver set)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    autoFrom := "From: " AccountLabel(SessionAccount)
    autoMenu.Add(autoFrom, (*) => 0), autoMenu.Disable(autoFrom)
    autoMenu.Add()
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: session account, aliases, statistics in the mail text
; optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname)
        , "[" pname "] Session " rest, BodyFromFile(path)
        , SessionAccount, AliasesOnSessions)
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp")
        return Trim(Opt(acc, "User")) != "" ? Trim(Opt(acc, "User")) : "(account not set up!)"
    return method " (its default account)"
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if (Trim(to) = "") {
        MsgBox "No receiver set.`nType one into the box next to Send, or enter your email address in the script (look for 'EMAIL HERE').", "Recorder", "Icon!"
        return false
    }
    if Mailing
        return false
    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    method := Opt(acc, "Method")
    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    try FileDelete attach                        ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

SendThunderbird(attach, subject, body, to) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(to, "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("to='{}',subject='{}',body='{}',attachment='{}'", to, subject, body, url)
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
SendOutlook(attach, subject, body, to) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        mail.To      := RegExReplace(to, "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        mail.Send()
        return true
    } catch as e {
        MsgBox "Outlook sending failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## What changed
- **Toolbar:** removed the `StatusTxt` text control.
- **`StartRec`, `BeginRecording`, `StopRec`, `SavePattern`:** removed the lines that updated the status text. Two small lines that only existed to fill in that text are gone too.
- **`SendMail`:** removed the "Sending…", "Sent: …" and "Sending failed" status updates, and the `from` line that only fed them.

## What you'll notice
- **You can still tell when it's recording:** the toolbar turns red for a manual recording and orange for an auto session, and the Start/Stop buttons switch on and off.
- **Errors still pop up:** if sending fails, you still get a message box saying why.
- **Successful sends are now silent:** the toolbar may freeze for a few seconds while sending, then nothing else happens. If you'd like a small Windows notification when a mail goes out, that's a one-line change.

The script is only a little shorter now, about 25 lines. That's fine, because it's still easy for me to work with in one piece. Tell me what's wrong with ShareAccount whenever you're ready.


### 👤 User (Message 49)

So when I manual send and it tells me ShareAccount is not set up yet and doesn't default to thunderbird or outlook. It should first try to default to thunderbird, as if i set it to "thunderbird" instead of "smtp" and then try to default to outlook. also when I do not have recipient set it should just open an email to an emtpy recipeint instead of giving me an error message "no receiver set."


### 🤖 Assistant (Message 50)

Both changes are in the script below:

- **When ShareAccount has no login:** the script now uses Thunderbird. If Thunderbird isn't installed, it uses classic Outlook. An error only appears if neither one is found.
- **When there's no receiver:** the email opens in Thunderbird or Outlook with the receiver left blank, so you can type it in and press Send yourself. This happens even if your SMTP login is set up, because SMTP can't open an email for you to edit.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

global Recording := false, Arming := false, Mailing := false
global AutoSession := false, AutoProfile := 0, SkipHwnd := 0
global TargetHwnd := 0, TargetWin := "", TargetExe := "", TargetClass := "", TargetTitle := ""
global Lines := [], Held := Map(), UsedKeys := Map(), LastTick := 0
global KeyCounts := Map(), ComboCounts := Map(), StartTick := 0, StartStamp := ""
global PrevActive := 0                                   ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until recording) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the file
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (Recording || Arming) ? StopRec() : StartRec()

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
StartRec() {
    global
    if (Recording || Arming || Mailing)
        return
    Arming := true
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    SetTimer ArmCheck, 100
}

ArmCheck() {
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    SetTimer ArmCheck, 0
    BeginRecording(hwnd)
}

BeginRecording(hwnd, profile := 0) {
    global
    TargetHwnd  := hwnd
    TargetWin   := "ahk_id " hwnd
    TargetExe   := WinGetProcessName(hwnd)
    TargetClass := WinGetClass(hwnd)
    TargetTitle := WinGetTitle(hwnd)
    AutoProfile := profile
    Lines := [], Held := Map(), UsedKeys := Map()
    KeyCounts := Map(), ComboCounts := Map()
    LastTick := StartTick := A_TickCount
    StartStamp := A_Now
    Arming := false, Recording := true, AutoSession := IsObject(profile)
    IH.Start()
    for h in MouseHotkeys
        Hotkey h, "On"
    BtnStart.Enabled := false, BtnStop.Enabled := true
    BtnSave.Enabled := false, BtnSend.Enabled := false
    G.BackColor := AutoSession ? "FFCC66" : "FF9999"
}

; stops the hooks and releases held keys (no saving)
EndCapture() {
    global Recording, AutoSession
    Recording := false, AutoSession := false
    IH.Stop()
    for h in MouseHotkeys
        Hotkey h, "Off"
    for k in Held.Clone()                        ; release anything still held when you stopped
        AddLine(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                   : 'Send "{Blind}{' EscKey(k) ' up}"')
    Held.Clear()
}

StopRec() {
    global Arming, SkipHwnd
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    G.BackColor := ""
    BtnStart.Enabled := true, BtnStop.Enabled := false
    BtnSave.Enabled := true, BtnSend.Enabled := true
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    if !wasAuto {                                ; normal manual recording
        WritePattern(LogFile)
        return
    }
    ; automatic session: own file name + own receiver
    SkipHwnd := TargetHwnd                       ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName()
    WritePattern(file)
    SendAuto(file)                               ; always the normal receivers (box is ignored here)
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

; ---------------- Automatic game sessions ----------------
AutoWatch() {
    global SkipHwnd
    if !AutoEnabled
        return
    if AutoSession {
        if (Recording && !WinExist("ahk_id " TargetHwnd))
            StopRec()                            ; game window closed -> save + send
        return
    }
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if (Recording || Arming || Mailing)
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        BeginRecording(hwnd, p)
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName() {
    game := RegExReplace(TargetExe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(AutoProfile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

ExitSave(*) {
    if !Recording
        return
    wasAuto := AutoSession
    EndCapture()
    WritePattern(wasAuto ? AutoFileName() : LogFile)   ; saved only, not sent (resend via Send menu)
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (Recording || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (Recording || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(empty - opens an email for you to fill in)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    autoFrom := "From: " AccountLabel(SessionAccount)
    autoMenu.Add(autoFrom, (*) => 0), autoMenu.Disable(autoFrom)
    autoMenu.Add()
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: session account, aliases, statistics in the mail text
; optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname)
        , "[" pname "] Session " rest, BodyFromFile(path)
        , SessionAccount, AliasesOnSessions)
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    if (Held.Has(k) || !WinActive(TargetWin))
        return                                   ; ignore auto-repeat & other windows
    Held[k] := true, UsedKeys[k] := true
    KeyCounts[k] := KeyCounts.Get(k, 0) + 1                          ; statistics
    if !IsModifier(k) && (mods := ModPrefix()) != ""
        ComboCounts[mods k] := ComboCounts.Get(mods k, 0) + 1        ; e.g. "Ctrl+C"
    AddLine('Send "{Blind}{' EscKey(k) ' down}"')
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    if !Held.Has(k)
        return
    Held.Delete(k)
    AddLine('Send "{Blind}{' EscKey(k) ' up}"')
}

MouseEvt(btn, dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (dir = "Down") {
        if (win = G.Hwnd || Held.Has(btn) || !WinActive(TargetWin))
            return
        Held[btn] := true, UsedKeys[btn] := true
    } else {
        if !Held.Has(btn)
            return
        Held.Delete(btn)
    }
    AddLine(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
}

WheelEvt(dir, *) {
    if !Recording
        return
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd || !WinActive(TargetWin))
        return
    AddLine(Format('Click "{} {} {}"', x, y, dir))
}

; ---------------- Helpers ----------------
AddLine(txt) {
    global LastTick
    now := A_TickCount
    if (Lines.Length && now - LastTick > 0)
        Lines.Push("Sleep " (now - LastTick))
    Lines.Push(txt)
    LastTick := now
}

KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

ModPrefix() {
    p := ""
    if (Held.Has("LControl") || Held.Has("RControl"))
        p .= "Ctrl+"
    if (Held.Has("LAlt") || Held.Has("RAlt"))
        p .= "Alt+"
    if (Held.Has("LShift") || Held.Has("RShift"))
        p .= "Shift+"
    if (Held.Has("LWin") || Held.Has("RWin"))
        p .= "Win+"
    return p
}

; ---------------- Statistics ----------------
StatsText() {
    total := 0
    for k, n in KeyCounts
        total += n
    secs   := (A_TickCount - StartTick) // 1000
    dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
    perMin := secs ? total * 60 / secs : 0
    s := "`n; ===================== Statistics =====================`n"
    s .= "; Profile: " (IsObject(AutoProfile) ? Opt(AutoProfile, "Name") : "manual recording") "`n"
    s .= "; Window:  " TargetTitle "   (" TargetExe ")`n"
    s .= "; Started: " FormatTime(StartStamp, "yyyy-MM-dd HH:mm") "`n"
    s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
    s .= "; (mouse clicks are not counted)`n"
    s .= "`n; ---- Keys (most used first) ----`n" CountTable(KeyCounts, total)
    if ComboCounts.Count
        s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(ComboCounts, total)
    return s
}

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}

WritePattern(path) {
    keys := ""
    for k in UsedKeys
        keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
    body := ""
    for l in Lines
        body .= "    " l "`n"

    tpl := "
    (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
    )"
    txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " TargetExe " ahk_class " TargetClass)
    txt := StrReplace(txt, "{{KEYS}}", keys)
    txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
    txt := StrReplace(txt, "{{BODY}}", body)
    txt := StrReplace(txt, "{{STATS}}", StatsText())
    try FileDelete path
    FileAppend txt, path, "UTF-8"
}
```

## What changed
- **`SendMail`:** this is where the new decision is made:
  - If the method is `"smtp"` but the login is empty, or there's no receiver, it switches to Thunderbird. If Thunderbird isn't installed, it switches to Outlook.
  - The "No receiver set" error is gone.
- **New `PickMailProgram()` and `OutlookInstalled()`:** these check what's installed, Thunderbird first and then classic Outlook.
- **`SendThunderbird` / `SendOutlook`:** both have a new "just open it" mode. They open the email with the subject, text and attachment already filled in, and leave the receiver field blank.
- **Send menu:**
  - The "From:" line now shows which program will actually be used, for example `thunderbird (its default account - smtp not set up)`.
  - When there's no receiver, the "To:" line now says `(empty - opens an email for you to fill in)`.

## Things to know
- **The fallback only happens if Thunderbird isn't installed.** If Thunderbird is installed but has a problem while sending, the script does *not* try Outlook next. If it did, you could end up sending the same email twice.
- **Auto sessions use the same rules.** If SessionAccount also has no login, auto sessions will go through Thunderbird or Outlook too, which sends them from that program's default account. If no receiver is set for auto sessions, an email will open for you when the game closes. Tell me if you'd rather auto sessions just save the file and not send anything in that case.
- **The temporary copy of the file is kept when an email is left open for you.** That's because Thunderbird only reads the attachment when you press Send. These copies sit in your Windows temp folder and get overwritten the next time you send that same file.


### 👤 User (Message 51)

Hot damn we are almost done, alright so when it autorecords I am unable to manually record. Can we make it so that the autorecord stuff for the statistics can work in parallel with the manual record for the patterns?


### 🤖 Assistant (Message 52)

Yes. Auto sessions now run in the background, and you can still Start and Stop manual recordings while one is running. Both can record the same game window at once, and each keeps its own key list, statistics and file. The full script is below. Only the recording parts changed; the email code is the same as before.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global AutoLbl  := G.Add("Text", "x+8 y9 w130", "")                       ; shows a running auto session
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the files
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    Write(path) {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
    AutoLbl.Value := IsObject(AutoRec) ? "● Auto: " Opt(AutoRec.Profile, "Name") : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; save the auto session + queue it for sending
StopAuto(send := true) {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName(s)
    s.Write(file)
    if send {
        PendingSends.Push(file)
        FlushPending()
    }
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> save + send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(s.Profile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

; closing the recorder: save both recordings (not sent - resend via Send menu)
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        s.Write(AutoFileName(s))
    }
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(empty - opens an email for you to fill in)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    autoFrom := "From: " AccountLabel(SessionAccount)
    autoMenu.Add(autoFrom, (*) => 0), autoMenu.Disable(autoFrom)
    autoMenu.Add()
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    if IsObject(AutoRec)
        m.Add("Stop auto session now (save + send)", (*) => StopAuto())
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: session account, aliases, statistics in the mail text
; optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname)
        , "[" pname "] Session " rest, BodyFromFile(path)
        , SessionAccount, AliasesOnSessions)
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}
```

## How it works now
- **Each recording keeps its own data.** Manual and auto recordings each have their own keys, timing and statistics. Starting or stopping a manual recording doesn't affect the auto session, and the reverse is also true.
- **Every key and click goes to both recordings.** It's only recorded if that recording's window is active, just like before.
- **Toolbar colors:**
  - **Red** means a manual recording is running. It doesn't matter whether an auto session is running too.
  - **Orange** means only an auto session is running.
  - A new label, such as **"● Auto: Modded"**, shows whenever an auto session is running.

## Things to know
- **Start, Stop and F9 only control manual recording now.** An auto session still ends when you close the game. To end one early, open the Send menu and choose **"Stop auto session now (save + send)"**. That option only appears while a session is running.
- **The Send menu works during an auto session.** It's still greyed out while a manual recording is running.
- **If you close the game during a manual recording,** the manual recording is saved to `~pattern.ahk` and the auto session is saved and sent, the same as before.
- **Auto-session emails no longer get lost.** Before, if you were already sending something when the game closed, the session email was quietly skipped. Now it waits and sends once the other email is done.


### 👤 User (Message 53)

ok, almost good. earlier we trimmed the gui and removed all the blah blah stuff about what was sent from the right side of the gui and now you made it bigger again. can you remove that gui part? the colors are more than sufficient


### 🤖 Assistant (Message 54)

Done. I removed the label, so the toolbar is back to its earlier size and the colors are the only signal. Two places in the script change.

### 1. Toolbar section
Delete the `AutoLbl` line. The buttons should now look like this:

```autohotkey
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
```

### 2. `UpdateToolbar()`
Replace the whole function with this. It's the same as before, minus the label line:

```autohotkey
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}
```

Nothing else changes. The colors mean:
- **Red:** a manual recording is running. An auto session may be running too.
- **Orange:** only an auto session is running.
- **Normal color:** nothing is recording.

If you're ever unsure whether an auto session is running during a manual recording, open the Send menu. **"Stop auto session now (save + send)"** only shows up while an auto session is running.


### 👤 User (Message 55)

Alrighty, now that we fixed up the gui I realize that we don't even need the AutoPatterns folder since we cannot interrupt the sending anymore. Can we just have a temp file instead like we do with ~pattern.ahk that gets removed after it is sent


### 🤖 Assistant (Message 56)

Yes. Since sending can't be interrupted anymore, there's no reason to keep auto patterns on disk. The auto session can use a temp file the same way `~pattern.ahk` does.

**One catch:** I can't see the current script in this conversation, so I'm going from the names we've used before. Where your code is different, I've marked it below. If something doesn't match, paste the auto-stop function and I'll give you an exact replacement.

### 1. Add a temp path for auto patterns
Put this next to your other globals. It uses a different name from `~pattern.ahk` because a manual and an auto recording can run at the same time, and they shouldn't overwrite each other's file.

```autohotkey
global AutoTmp := A_ScriptDir "\~autopattern.ahk"
```

### 2. Remove the AutoPatterns folder code
Delete these:
- The `DirCreate(... "\AutoPatterns")` line, usually near the top or in the auto-start code
- The line that builds the timestamped filename, for example `path := A_ScriptDir "\AutoPatterns\" FormatTime(...) ".ahk"`
- Any cleanup or pruning code that loops over the AutoPatterns folder, if you have it

### 3. Write, send, delete
In the function that ends the auto session and sends the pattern, write to `AutoTmp` instead of the old path. Then delete the file after sending:

```autohotkey
    try FileDelete(AutoTmp)                 ; clear any leftover
    FileAppend(code, AutoTmp, "UTF-8")      ; 'code' = the generated pattern text (use your variable name)
    SendPattern(AutoTmp, recv)              ; your existing send call, unchanged
    try FileDelete(AutoTmp)
```

`SendPattern` and `code` stand in for whatever names your script uses. Keep your existing send line and only change the path it gets.

### 4. Clean up leftovers at startup (optional)
If the script crashes or gets killed in the middle of a send, the temp file could be left behind. Add this near the top so it's cleared on the next launch:

```autohotkey
try FileDelete(A_ScriptDir "\~autopattern.ahk")
```

After that you can delete the old `AutoPatterns` folder by hand.


### 👤 User (Message 57)

I am not very good at this, I will just send you the code again so you can do those things. I have also not done the change from the previous promt 1. Toolbar section and 2. UpdateToolbar(), do those aswell.

#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions (and resending them)          -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for file name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global AutoSaveDir := A_ScriptDir "\AutoPatterns"        ; automatic sessions (never overwritten)
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
DirCreate AutoSaveDir
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global AutoLbl  := G.Add("Text", "x+8 y9 w130", "")                       ; shows a running auto session
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves the files
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    Write(path) {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
    AutoLbl.Value := IsObject(AutoRec) ? "● Auto: " Opt(AutoRec.Profile, "Name") : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; save the auto session + queue it for sending
StopAuto(send := true) {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    file := AutoFileName(s)
    s.Write(file)
    if send {
        PendingSends.Push(file)
        FlushPending()
    }
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> save + send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profileName) {
    for p in AutoProfiles
        if (SafeName(Opt(p, "Name")) = profileName && Trim(Opt(p, "EmailTo")) != "")
            return Trim(Opt(p, "EmailTo"))
    return AutoRecipient()
}

; e.g. AutoPatterns\Modded_factorio_2026-11-18_20.15.30.ahk
AutoFileName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return AutoSaveDir "\" SafeName(Opt(s.Profile, "Name")) "_" game "_"
         . FormatTime(, "yyyy-MM-dd_HH.mm.ss") ".ahk"
}

; closing the recorder: save both recordings (not sent - resend via Send menu)
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        s.Write(AutoFileName(s))
    }
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(empty - opens an email for you to fill in)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    autoMenu := Menu(), autoCount := 0
    autoFrom := "From: " AccountLabel(SessionAccount)
    autoMenu.Add(autoFrom, (*) => 0), autoMenu.Disable(autoFrom)
    autoMenu.Add()
    Loop Files AutoSaveDir "\*.ahk" {
        autoMenu.Add(A_LoopFileName, SendAuto.Bind(A_LoopFileFullPath, override))
        autoCount++
    }
    m.Add()
    if autoCount
        m.Add("Auto sessions (resend)", autoMenu)
    if IsObject(AutoRec)
        m.Add("Stop auto session now (save + send)", (*) => StopAuto())
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Add("Open AutoPatterns folder", (*) => Run(AutoSaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: session account, aliases, statistics in the mail text
; optional receiver (from the box); "" = profile / default receiver
SendAuto(path, to := "", *) {
    SplitPath path, , , , &noExt
    parts := StrSplit(noExt, "_", , 2)           ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    SendMail(path, to != "" ? to : RecipientFor(pname)
        , "[" pname "] Session " rest, BodyFromFile(path)
        , SessionAccount, AliasesOnSessions)
}

; takes the statistics block from the end of a pattern file as email text
BodyFromFile(path) {
    txt := FileRead(path, "UTF-8")
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

SendMail(path, to, subject, body, acc, useAliases) {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}


### 🤖 Assistant (Message 58)

I've made the AutoPatterns change in the full script below. You can replace your whole file with it.

**I couldn't make the toolbar changes.** I can't see the earlier message where I described "1. Toolbar section" and "2. UpdateToolbar()", so I don't know exactly what they said. I left the toolbar section and `UpdateToolbar()` exactly as they are in your file. If you paste those two instructions here, I'll add them and send the full script again.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global AutoLbl  := G.Add("Text", "x+8 y9 w130", "")                       ; shows a running auto session
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
    AutoLbl.Value := IsObject(AutoRec) ? "● Auto: " Opt(AutoRec.Profile, "Name") : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(empty - opens an email for you to fill in)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    if IsObject(AutoRec)
        m.Add("Stop auto session now (send)", (*) => StopAuto())
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}
```

## What changed

- **No more AutoPatterns folder.** When an auto session ends, it's written to `~autopattern.ahk`, emailed, and then the file is deleted. If the script crashed before, any leftover file is removed at startup.
- **The attachment name stays the same.** Emails still arrive with an attachment like `Modded_factorio_2026-11-18_20.15.30.ahk` and the same subject as before.
- **Queued sessions are saved right away.** If an email is already being sent when a session ends, the finished session's text is kept in memory until it can be sent. This keeps the duration and statistics correct.
- **Closing the recorder now sends the auto session.** Before, it was only saved to the folder. Since there's no folder anymore, it's emailed before the script exits.
- **Send menu:** The "Auto sessions (resend)" submenu and "Open AutoPatterns folder" are gone. "Stop auto session now" is still there.

**Note:** Since auto sessions aren't kept on disk anymore, a failed send (no internet, wrong password) means that session is lost. You'll still get the error message.

After you've tested it, you can delete the old `AutoPatterns` folder by hand. And remember to paste those two toolbar instructions so I can finish that part.


### 👤 User (Message 59)

I wrote:
ok, almost good. earlier we trimmed the gui and removed all the blah blah stuff about what was sent from the right side of the gui and now you made it bigger again. can you remove that gui part? the colors are more than sufficient

and then you wrote:
Done. I removed the label, so the toolbar is back to its earlier size and the colors are the only signal. Two places in the script change.

### 1. Toolbar section
Delete the `AutoLbl` line. The buttons should now look like this:

```autohotkey
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
```

### 2. `UpdateToolbar()`
Replace the whole function with this. It's the same as before, minus the label line:

```autohotkey
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}
```

Nothing else changes. The colors mean:
- **Red:** a manual recording is running. An auto session may be running too.
- **Orange:** only an auto session is running.
- **Normal color:** nothing is recording.

If you're ever unsure whether an auto session is running during a manual recording, open the Send menu. **"Stop auto session now (save + send)"** only shows up while an auto session is running.


### 🤖 Assistant (Message 60)

Thanks for pasting that. Both toolbar changes are now in the full script below, along with the AutoPatterns change from before. You can replace your whole file with it.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(empty - opens an email for you to fill in)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    if IsObject(AutoRec)
        m.Add("Stop auto session now (send)", (*) => StopAuto())
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}
```

## What I changed

- **Toolbar section:** I removed the `AutoLbl` line, so the toolbar is back to its smaller size. The X button now sits right after Send.
- **`UpdateToolbar()`:** I removed the line that wrote text into the label. Only the colors show the status now:
  - **Red:** A manual recording is running. An auto session may be running too.
  - **Orange:** Only an auto session is running.
  - **Normal color:** Nothing is recording.

**One small fix to my earlier tip:** the Send menu option is now called **"Stop auto session now (send)"**, not "(save + send)". Sessions aren't saved to a folder anymore. Like before, you only see this option while an auto session is running. That's a quick way to check if one is running during a red (manual) recording.


### 👤 User (Message 61)

Actually I don't think I will ever use that Stop auto session thing, can you trim that so the code gets smaller, honestly a bunch of things in the send menu could be removed, the to field doesn't need to be there since we set it with the receiver field anyways and the from field is also pointless since thunderbird pops up. let's trim that code :D


### 🤖 Assistant (Message 62)

Done. The Send menu now lists only your patterns and the "Open SavedPatterns folder" option. You'll need to change two things.

### 1. Replace the whole `SendMenu()` function

```autohotkey
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}
```

### 2. Delete the `AccountLabel()` function

Only the "From" line used it, so you can delete this whole block:

```autohotkey
; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}
```

## Don't delete these

- **`StopAuto()`:** This doesn't depend on the menu. It runs on its own when you close the game window, and that's how auto sessions get sent.
- **`PickMailProgram()` and `OutlookInstalled()`:** `SendMail()` still uses these to switch to Thunderbird or Outlook when there's no SMTP login.

The receiver box and invalid-address check work exactly as before.

**One thing that changes:** without the "Stop auto session" option, the menu can't tell you anymore whether an auto session is running during a manual recording. The toolbar is red whenever a manual recording runs, so you'll only see orange when an auto session runs on its own. If you don't need to know that, nothing is lost.


### 👤 User (Message 63)

I will copy the whole code again so you can make the changes:
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    ; show sender AND receiver before anything is sent
    fromTxt := "From: " AccountLabel(ShareAccount)
    toTxt   := "To: " (Trim(to) != "" ? to : "(empty - opens an email for you to fill in)")
    m.Add(fromTxt, (*) => 0), m.Disable(fromTxt)
    m.Add(toTxt, (*) => 0),   m.Disable(toTxt)
    m.Add()

    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    if IsObject(AutoRec)
        m.Add("Stop auto session now (send)", (*) => StopAuto())
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; short text for menus: who the mail comes from
AccountLabel(acc) {
    method := Opt(acc, "Method")
    if (method = "smtp") {
        if SmtpConfigured(acc)
            return Trim(Opt(acc, "User"))
        p := PickMailProgram()                   ; login empty -> fallback program
        return p != "" ? p " (its default account - smtp not set up)" : "(nothing set up!)"
    }
    return method " (its default account)"
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}


### 🤖 Assistant (Message 64)

Here's the full script with both changes made:

- **`SendMenu()`** no longer has the From, To and "Stop auto session now" lines.
- **`AccountLabel()`** is deleted, since only the From line used it.

The rest of the script is exactly the same.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it (runs when the game window closes)
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}
```

I also updated the comment above `StopAuto()` to say it runs when the game window closes. That's now the only way an auto session ends, apart from closing the recorder itself, which sends it too.


### 👤 User (Message 65)

Very nice, looks good. I've noticed though that my tray gets filled up quite a bit with all the patterns now. Can we make it so that the script and the patterns made by the script don't overload my tray?


### 🤖 Assistant (Message 66)

I made two changes:

1. **The recorder has no tray icon anymore.** You close it with the **X** on the toolbar, as before.
2. **New patterns have no tray icon, and only one runs at a time.** When you start a pattern, it closes any other pattern that's still running. This also fixes a hidden problem: before, if several patterns were open, pressing F8 in the game started **all of them at once**.

Since patterns no longer have a tray icon, you close them with **F10**:

| Key | While the pattern is playing | While it's waiting |
|---|---|---|
| **F8** | (ignored) | Start the pattern |
| **F10** | Stop and stay ready for F8 | **Close the pattern** |

The "Pattern ready" message is now a small tooltip that goes away after 4 seconds. Windows doesn't show tray notifications for scripts without a tray icon, so a tooltip is needed instead.

### Change 1: top of the script

Add `#NoTrayIcon` to the first lines:

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window
```

### Change 2: replace the whole `Text()` method inside `class RecSession`

```autohotkey
    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; only ONE recorded pattern runs at a time -> close any other one
DetectHiddenWindows true
for h in WinGetList("RecorderPattern ahk_class AutoHotkey")
    if (h != A_ScriptHwnd)
        WinClose h
WinSetTitle "RecorderPattern", A_ScriptHwnd
DetectHiddenWindows false

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort
Playing  := false

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; while playing: stop  |  while waiting: close this pattern

OnExit((*) => ReleaseAll())
ToolTip "Pattern ready: switch to the game and press F8   (F10 = close)"
SetTimer () => ToolTip(), -4000

Play() {
    global Playing
    if Playing
        return
    Playing := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    Playing := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    if Playing {
        Reload                  ; stop playing, stay ready for F8
        return
    }
    ExitApp                     ; not playing -> close the pattern
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }
```

### Patterns you already have

Only **new recordings** get this behavior. Patterns already in SavedPatterns, or ones friends sent you earlier, still have a tray icon. They also stay running after F10 and don't close each other.

- **To clear them now:** right-click their tray icons and choose **Exit**.
- **To update a saved pattern:** open it in Notepad. Replace everything above the `; ===================== Recorded pattern =====================` line with the matching part of the new template. Fill in `GameWin` and `UsedKeys` with the values from your old file. The recorded part below that line can stay as it is.


### 👤 User (Message 67)

Nice, can you also make it so that I can play fullscreen without the gui. can we make it so that pressing the x button hides the gui


### 🤖 Assistant (Message 68)

The X button now hides the toolbar instead of closing the recorder. Since the recorder no longer has a tray icon, I added hotkeys to bring the toolbar back and to close the recorder.

| Action | What it does |
|---|---|
| **X** on the toolbar | Hide the toolbar (the recorder keeps running) |
| **Ctrl+F12** | Show or hide the toolbar |
| **Shift + click X** or **Ctrl+Shift+F12** | Close the recorder completely |

### Step 1: find your GUI's name and replace the X handler

Look for the line where the toolbar is created. It looks something like `RecGui := Gui(...)`. The name before `:=` is your GUI's name. **In the code below, replace every `RecGui` with that name.**

Then find the line that makes the X close the recorder. It probably looks like this:

```autohotkey
RecGui.OnEvent("Close", (*) => ExitApp())
```

Replace it with:

```autohotkey
RecGui.OnEvent("Close", HideToolbar)   ; X = hide instead of close
```

### Step 2: paste this at the very bottom of the script

Put it outside any `class` block. If the script uses `#HotIf` anywhere, keep the `#HotIf` line at the top of this block. It makes sure the hotkeys work in every window.

```autohotkey
; ===================== Show / hide the toolbar =====================
#HotIf
^F12::ToggleToolbar()       ; Ctrl+F12       = show / hide the toolbar
^+F12::ExitApp              ; Ctrl+Shift+F12 = close the recorder completely

HideToolbar(*) {
    if GetKeyState("Shift") {       ; Shift + X = really close the recorder
        ExitApp
    }
    RecGui.Hide()
    ToolTip "Recorder hidden - Ctrl+F12 shows it again"
    SetTimer () => ToolTip(), -3000
    return true                     ; we handled it, don't close
}

ToggleToolbar() {
    if DllCall("IsWindowVisible", "Ptr", RecGui.Hwnd)
        HideToolbar()
    else
        RecGui.Show("NoActivate")   ; show again without pulling focus from the game
}
```

### How it works

- **Hiding doesn't stop anything.** If a recording is running, it keeps going while the toolbar is hidden. You can record your full-screen gameplay normally.
- **`NoActivate`** shows the toolbar again without taking focus away from the game. The toolbar also comes back in the same spot where you left it.
- **Using Ctrl+F12 while recording:** depending on how the recording is captured, Ctrl+F12 may get recorded too. If it does, delete that line from the saved pattern. Or hide the toolbar before you start recording.

### About full-screen mode

If your game runs in **exclusive full-screen**, showing the toolbar over it (especially if it's set to always-on-top) can make Windows minimize the game or flicker. If that happens, switch the game to **borderless windowed** (sometimes called "windowed full-screen"). It looks the same, and the toolbar can show and hide without problems. The patterns work either way, because they use coordinates relative to the game window.


### 👤 User (Message 69)

I will paste the whole thing again so you can make the changes of the last 2 prompts.

#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  (*) => ExitApp())
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9:: (IsObject(ManualRec) || Arming) ? StopRec() : StartRec()   ; manual recording only

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it (runs when the game window closes)
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}


### 🤖 Assistant (Message 70)

I added the hide-toolbar change to your script. I can only see one earlier request in this chat (the full-screen / hide-the-toolbar one), so that's the only change included. If you meant another change from before that, paste it or describe it and I'll add it.

Your toolbar is called `G`, and its X is a button rather than a window close button. So I connected the hide action straight to the X button. I also changed three other things to make playing full-screen easier.

## What's new

| Action | What it does |
|---|---|
| **X** on the toolbar | Hides the toolbar. Recordings keep running. |
| **Ctrl+F12** | Shows or hides the toolbar. It comes back in the same spot without taking focus from the game. |
| **Shift + click X** or **Ctrl+Shift+F12** | Closes the recorder. It still saves the manual recording and sends the auto session. |
| **F9 while the toolbar is hidden** | Plays a short beep: high = recording started, low = recording stopped. You can turn this off with `BeepWhenHidden := false`. |

- **F12 isn't recorded**, the same way F9 already wasn't. When you press Ctrl+F12 during a recording, only the Ctrl press may show up in the pattern. You can delete that from the pattern or leave it, since a Ctrl press on its own does nothing.

All changed lines are marked with `; NEW`.

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- TOOLBAR ----                                                      ; NEW
; X = hide toolbar   Ctrl+F12 = show/hide   Shift+X or Ctrl+Shift+F12 = close recorder
global BeepWhenHidden := true             ; NEW: beep on F9 start/stop while the toolbar is hidden

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  HideToolbar)   ; NEW: X = hide (Shift+X = close the recorder)
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.KeyOpt("{F12}", "-N")             ; NEW: don't record the show/hide toolbar hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9::ToggleManual()                   ; NEW: manual recording only (beeps while the toolbar is hidden)
^F12::ToggleToolbar()                ; NEW: Ctrl+F12       = show / hide the toolbar
^+F12::ExitApp                       ; NEW: Ctrl+Shift+F12 = close the recorder completely

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; abort at any time

OnExit((*) => ReleaseAll())
TrayTip "Switch to the game and press F8   (F10 = abort)", "Pattern ready"

Play() {
    static running := false
    if running
        return
    running := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    running := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    Reload
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Show / hide the toolbar ----------------              ; NEW (whole section)
ToolbarVisible() => DllCall("IsWindowVisible", "Ptr", G.Hwnd)

; X button: hide (recordings keep running). Shift + X = close the recorder.
HideToolbar(*) {
    if GetKeyState("Shift") {
        ExitApp                                  ; OnExit still saves / sends
    }
    G.Hide()
    ToolTip "Recorder hidden - Ctrl+F12 shows it again"
    SetTimer () => ToolTip(), -3000
}

ToggleToolbar() {
    if ToolbarVisible()
        HideToolbar()
    else
        G.Show("NoActivate")                     ; same spot, doesn't take focus from the game
}

; F9: start/stop manual recording - beeps when you can't see the toolbar
ToggleManual() {
    if (IsObject(ManualRec) || Arming) {
        StopRec()
        HiddenBeep(500)                          ; low beep = stopped
    } else {
        StartRec()
        if Arming
            HiddenBeep(1000)                     ; high beep = started
    }
}

HiddenBeep(freq) {
    if (BeepWhenHidden && !ToolbarVisible())
        SoundBeep freq, 80
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it (runs when the game window closes)
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}
```

**Playing full-screen:** you can hide the toolbar, start the game, and press F9 to start or stop recording. Automatic sessions still start by themselves. If the game uses **exclusive full-screen**, Windows may minimize it or make it flicker when the toolbar or the "Recorder hidden" message appears on top. Switching the game to **borderless windowed** avoids this. Recordings work the same either way.


### 👤 User (Message 71)

The other change was the following:
I wrote: Very nice, looks good. I've noticed though that my tray gets filled up quite a bit with all the patterns now. Can we make it so that the script and the patterns made by the script don't overload my tray?

and you wrote:
I made two changes:

1. **The recorder has no tray icon anymore.** You close it with the **X** on the toolbar, as before.
2. **New patterns have no tray icon, and only one runs at a time.** When you start a pattern, it closes any other pattern that's still running. This also fixes a hidden problem: before, if several patterns were open, pressing F8 in the game started **all of them at once**.

Since patterns no longer have a tray icon, you close them with **F10**:

| Key | While the pattern is playing | While it's waiting |
|---|---|---|
| **F8** | (ignored) | Start the pattern |
| **F10** | Stop and stay ready for F8 | **Close the pattern** |

The "Pattern ready" message is now a small tooltip that goes away after 4 seconds. Windows doesn't show tray notifications for scripts without a tray icon, so a tooltip is needed instead.

### Change 1: top of the script

Add `#NoTrayIcon` to the first lines:

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window
```

### Change 2: replace the whole `Text()` method inside `class RecSession`

```autohotkey
    ; builds the full pattern script as text
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; only ONE recorded pattern runs at a time -> close any other one
DetectHiddenWindows true
for h in WinGetList("RecorderPattern ahk_class AutoHotkey")
    if (h != A_ScriptHwnd)
        WinClose h
WinSetTitle "RecorderPattern", A_ScriptHwnd
DetectHiddenWindows false

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort
Playing  := false

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; while playing: stop  |  while waiting: close this pattern

OnExit((*) => ReleaseAll())
ToolTip "Pattern ready: switch to the game and press F8   (F10 = close)"
SetTimer () => ToolTip(), -4000

Play() {
    global Playing
    if Playing
        return
    Playing := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    Playing := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    if Playing {
        Reload                  ; stop playing, stay ready for F8
        return
    }
    ExitApp                     ; not playing -> close the pattern
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }
```

### Patterns you already have

Only **new recordings** get this behavior. Patterns already in SavedPatterns, or ones friends sent you earlier, still have a tray icon. They also stay running after F10 and don't close each other.

- **To clear them now:** right-click their tray icons and choose **Exit**.
- **To update a saved pattern:** open it in Notepad. Replace everything above the `; ===================== Recorded pattern =====================` line with the matching part of the new template. Fill in `GameWin` and `UsedKeys` with the values from your old file. The recorded part below that line can stay as it is.


### 🤖 Assistant (Message 72)

Here's the full script with both changes: the **hide-toolbar** change and the **tray-icon** change. Lines that changed are marked `; NEW`.

One thing changed from what I said in the tray answer: **X now hides the toolbar instead of closing the recorder.** Since the recorder has no tray icon, you close it like this:

| Action | What it does |
|---|---|
| **X** | Hides the toolbar. Recordings keep running. |
| **Ctrl+F12** | Shows or hides the toolbar. |
| **Shift + click X** or **Ctrl+Shift+F12** | Closes the recorder. It still saves the manual recording and sends the auto session. |

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon                          ; NEW: no tray icon for the recorder
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- TOOLBAR ----                                                      ; NEW
; X = hide toolbar   Ctrl+F12 = show/hide   Shift+X or Ctrl+Shift+F12 = close recorder
; (there is no tray icon - the recorder is closed with Shift+X or Ctrl+Shift+F12)
global BeepWhenHidden := true             ; NEW: beep on F9 start/stop while the toolbar is hidden

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  HideToolbar)   ; NEW: X = hide (Shift+X = close the recorder)
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.KeyOpt("{F12}", "-N")             ; NEW: don't record the show/hide toolbar hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9::ToggleManual()                   ; NEW: manual recording only (beeps while the toolbar is hidden)
^F12::ToggleToolbar()                ; NEW: Ctrl+F12       = show / hide the toolbar
^+F12::ExitApp                       ; NEW: Ctrl+Shift+F12 = close the recorder completely

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    ; NEW: patterns have no tray icon, only one runs at a time, F10 closes a waiting pattern
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; only ONE recorded pattern runs at a time -> close any other one
DetectHiddenWindows true
for h in WinGetList("RecorderPattern ahk_class AutoHotkey")
    if (h != A_ScriptHwnd)
        WinClose h
WinSetTitle "RecorderPattern", A_ScriptHwnd
DetectHiddenWindows false

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort
Playing  := false

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; while playing: stop  |  while waiting: close this pattern

OnExit((*) => ReleaseAll())
ToolTip "Pattern ready: switch to the game and press F8   (F10 = close)"
SetTimer () => ToolTip(), -4000

Play() {
    global Playing
    if Playing
        return
    Playing := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    Playing := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    if Playing {
        Reload                  ; stop playing, stay ready for F8
        return
    }
    ExitApp                     ; not playing -> close the pattern
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Show / hide the toolbar ----------------              ; NEW (whole section)
ToolbarVisible() => DllCall("IsWindowVisible", "Ptr", G.Hwnd)

; X button: hide (recordings keep running). Shift + X = close the recorder.
HideToolbar(*) {
    if GetKeyState("Shift") {
        ExitApp                                  ; OnExit still saves / sends
    }
    G.Hide()
    ToolTip "Recorder hidden - Ctrl+F12 shows it again"
    SetTimer () => ToolTip(), -3000
}

ToggleToolbar() {
    if ToolbarVisible()
        HideToolbar()
    else
        G.Show("NoActivate")                     ; same spot, doesn't take focus from the game
}

; F9: start/stop manual recording - beeps when you can't see the toolbar
ToggleManual() {
    if (IsObject(ManualRec) || Arming) {
        StopRec()
        HiddenBeep(500)                          ; low beep = stopped
    } else {
        StartRec()
        if Arming
            HiddenBeep(1000)                     ; high beep = started
    }
}

HiddenBeep(freq) {
    if (BeepWhenHidden && !ToolbarVisible())
        SoundBeep freq, 80
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it (runs when the game window closes)
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}
```

## Good to know

- **If the toolbar is hidden, nothing on screen shows the recorder is running,** because there's no tray icon either. Press **Ctrl+F12** to bring the toolbar back. Starting the recorder script again also works: `#SingleInstance Force` replaces the old copy, and the new one shows the toolbar.
- **How patterns work now:** **F8** starts a pattern in the game. **F10** stops it while it's playing, or closes it while it's waiting. Starting a new pattern closes any older one that's still running.
- **Patterns you already have** (older saved ones or ones friends sent) still have a tray icon and the old behavior. Right-click their tray icons and choose **Exit** to close them. To update one, replace the top part as I described before.


### 👤 User (Message 73)

ok, we are almost done here. 3 things. 1. I do not need the tooltip you added when pressing x, I do not need the hotkey to bring the hud back nor do i need the hotkey to completely stop the program and i also do not need the orange color when an autopattern starts recording. instead make regular recording orange. that is actually 4 things.


### 🤖 Assistant (Message 74)

Got it. Keep **X** as “hide the HUD,” but remove its tooltip and the extra hotkeys. Make manual recording orange; automatic sessions won’t change the HUD color.

Make these edits in the script:

1. **Remove** the `BeepWhenHidden` setting and the `IH.KeyOpt("{F12}", "-N")` line.
2. **Remove** these hotkeys:
   ```autohotkey
   ^F12::ToggleToolbar()
   ^+F12::ExitApp
   ```
3. Replace `HideToolbar` with this, and remove the `ToolbarVisible`, `ToggleToolbar`, and `HiddenBeep` functions:
   ```autohotkey
   HideToolbar(*) {
       G.Hide()
   }
   ```
4. Replace `ToggleManual()` with this version so F9 starts and stops recording without beeps:
   ```autohotkey
   ToggleManual() {
       if (IsObject(ManualRec) || Arming)
           StopRec()
       else
           StartRec()
   }
   ```
5. In `UpdateToolbar()`, replace the color assignment with:
   ```autohotkey
   G.BackColor := (IsObject(ManualRec) || Arming) ? "FFCC66" : ""
   ```

The toolbar’s **X will now hide it silently**. With no show-again hotkey, it will stay hidden until the script is restarted. The script still has no tray icon; without an exit hotkey, use Task Manager to close it. **F9** remains the manual start/stop key.


### 👤 User (Message 75)

Can you add all the changes you proposed to the code:

#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon                          ; NEW: no tray icon for the recorder
InstallKeybdHook
InstallMouseHook
CoordMode "Mouse", "Client"          ; mouse coords relative to the game window

; ======================= SETTINGS =======================
; EMAIL HERE
global EmailTo     := ""                  ; default receivers for the Send menu (when the box is empty)
                                          ; (several receivers: separate with , or ;)
                                          ; empty everywhere = an email opens for you to fill in
global AttachAsTxt := false               ; true = send as "name.ahk.txt" (if a mail filter blocks .ahk)

; ---- TOOLBAR ----                                                      ; NEW
; X = hide toolbar   Ctrl+F12 = show/hide   Shift+X or Ctrl+Shift+F12 = close recorder
; (there is no tray icon - the recorder is closed with Shift+X or Ctrl+Shift+F12)
global BeepWhenHidden := true             ; NEW: beep on F9 start/stop while the toolbar is hidden

; ---- SENDING ACCOUNTS ----
; ShareAccount   = patterns you send from the Send menu (to friends)      -> your REGULAR email
; SessionAccount = automatic game sessions                               -> your AUTOMATIC email
;   Method:   "smtp"        = send directly with the login below
;                             (login empty -> uses Thunderbird, or else Outlook, automatically)
;             "thunderbird" / "outlook" = use that program (it sends from ITS default account)
;   Server / Port: Gmail = "smtp.gmail.com" / 587
;   User:     full email address of that account (this is also the "From" address)
;   Password: Gmail = 16-char app password OF THAT ACCOUNT, no spaces
global ShareAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}
global SessionAccount := {
    Method:   "smtp",
    Server:   "smtp.gmail.com",
    Port:     587,
    User:     "",
    Password: ""
}

; ---- Automatic game sessions ----
global AutoEnabled := true                               ; false = turn auto recording off completely
global AutoEmailTo := ""                                 ; default receiver for auto sessions ("" = use EmailTo)

; One line per trigger. Checked top to bottom - the FIRST matching profile wins.
;   Name    = label for attachment name + email subject (e.g. "Modded")
;   Exe     = program file of the game ("" = any program)
;   Title   = what the window title must contain (RegEx, case-insensitive, "" = any title)
;             note: a dot must be written as  \.   ->  "1\.1\." means "1.1."
;   EmailTo = receiver for this profile ("" = use AutoEmailTo)
global AutoProfiles := [
    {Name: "Modded",   Exe: "factorio.exe", Title: "^Factorio.* 1\.1\.", EmailTo: ""},
    {Name: "Vanilla",  Exe: "factorio.exe", Title: "^Factorio.* 2\.0\.", EmailTo: ""},
    {Name: "Factorio", Exe: "factorio.exe", Title: "",                   EmailTo: ""}
]
; (the last one is a catch-all for any other version - delete it if you don't want that)

; ---- EMAIL PRIVACY ALIASES ----
; Format:  real text = what the email shows instead
; Case-insensitive, plain text (no RegEx). Longer entries are replaced first.
; Lines starting with ; are ignored.
global EmailAliases := "
(
; factorio = Game1
)"
global EmailAliasList := ParseAliases(EmailAliases)

global AliasesOnShared   := false    ; Send menu patterns: false = real names (what your friends need)
global AliasesOnSessions := true     ; automatic sessions:  true  = replace names with the aliases
global AnonymizeAttachment := true   ; when aliases are used: also clean the text INSIDE the attached .ahk

global LogFile     := A_ScriptDir "\~pattern.ahk"        ; current manual recording (overwritten each time)
global AutoTmp     := A_ScriptDir "\~autopattern.ahk"    ; auto session temp file (deleted after sending)
global SaveDir     := A_ScriptDir "\SavedPatterns"       ; your saved patterns
global MailTempDir := A_Temp "\RecorderMail"             ; temp copies for sending
DirCreate SaveDir
try FileDelete AutoTmp                                   ; leftover from a crash -> remove
; ========================================================

; two independent recordings that can run at the same time
global ManualRec := 0                ; manual pattern recording (Start/Stop/F9)
global AutoRec   := 0                ; automatic game session (statistics)
global Arming := false, Mailing := false, SkipHwnd := 0
global PendingSends := []            ; finished auto sessions waiting to be emailed
global PrevActive := 0               ; window to return to after typing a receiver

; ---------------- GUI (toolbar at top of screen) ----------------
; +E0x08000000 = WS_EX_NOACTIVATE -> clicking the toolbar does NOT steal focus from the game
; colors: red = manual recording, orange = only auto session, normal = nothing recording
global G := Gui("+AlwaysOnTop -Caption +ToolWindow +Border +E0x08000000", "Recorder")
G.SetFont("s9", "Segoe UI")
global BtnStart := G.Add("Button", "x5 y5 w55", "Start")
global BtnStop  := G.Add("Button", "x+5 w55 Disabled", "Stop")
global BtnEdit  := G.Add("Button", "x+5 w55", "Edit")
global BtnSave  := G.Add("Button", "x+5 w55", "Save")
global RecvEdit := G.Add("Edit", "x+5 yp+1 w180 h21 -Multi")              ; receiver override
global BtnSend  := G.Add("Button", "x+5 y5 w55", "Send")
global BtnExit  := G.Add("Button", "x+5 y5 w25", "X")
SendMessage(0x1501, 1, StrPtr("Receiver (empty = default)"), RecvEdit)   ; grey placeholder text
BtnStart.OnEvent("Click", (*) => StartRec())
BtnStop.OnEvent("Click",  (*) => StopRec())
BtnEdit.OnEvent("Click",  (*) => EditFile())
BtnSave.OnEvent("Click",  (*) => SavePattern())
BtnSend.OnEvent("Click",  (*) => SendMenu())
BtnExit.OnEvent("Click",  HideToolbar)   ; NEW: X = hide (Shift+X = close the recorder)
G.Show("NoActivate y0")              ; x omitted -> centered horizontally
OnMessage(0x201, ToolbarClick)       ; drag toolbar by background + focus the receiver box
OnMessage(0x6, ToolbarActivate)      ; give "don't steal focus" back when you leave the toolbar

; ---------------- Keyboard hook (shared by both recordings) ----------------
global IH := InputHook("V L0 I")     ; V = keys still reach the game, L0 = no text buffer, I = ignore AHK-sent keys
IH.KeyOpt("{All}", "N")              ; notify on every key down/up
IH.KeyOpt("{F9}", "-N")              ; don't record the Start/Stop hotkey
IH.KeyOpt("{F12}", "-N")             ; NEW: don't record the show/hide toolbar hotkey
IH.OnKeyDown := KeyDown
IH.OnKeyUp   := KeyUp

; ---------------- Mouse hotkeys (off until something records) ----------------
global MouseHotkeys := []
for b in ["LButton", "RButton", "MButton", "XButton1", "XButton2"] {
    Hotkey "~*" b, MouseEvt.Bind(b, "Down"), "Off"
    Hotkey "~*" b " Up", MouseEvt.Bind(b, "Up"), "Off"
    MouseHotkeys.Push("~*" b, "~*" b " Up")
}
for w in ["WheelUp", "WheelDown"] {
    Hotkey "~*" w, WheelEvt.Bind(w), "Off"
    MouseHotkeys.Push("~*" w)
}

OnExit(ExitSave)                     ; closing the recorder mid-session still saves / sends
SetTimer AutoWatch, 1000             ; check once per second for a matching game window

F9::ToggleManual()                   ; NEW: manual recording only (beeps while the toolbar is hidden)
^F12::ToggleToolbar()                ; NEW: Ctrl+F12       = show / hide the toolbar
^+F12::ExitApp                       ; NEW: Ctrl+Shift+F12 = close the recorder completely

; while typing in the receiver box
#HotIf WinActive("ahk_id " G.Hwnd)
Enter::ReturnFocus()                             ; done typing -> back to the previous window
Esc:: (RecvEdit.Value := "", ReturnFocus())      ; clear box -> back to default receivers
#HotIf

; =====================================================================
; One recording = one RecSession object. Manual and auto each have their own,
; so they never mix keys, statistics or files.
class RecSession {
    __New(hwnd, profile := 0) {
        this.Hwnd       := hwnd
        this.Win        := "ahk_id " hwnd
        this.Exe        := WinGetProcessName(hwnd)
        this.WinClass   := WinGetClass(hwnd)
        this.Title      := WinGetTitle(hwnd)
        this.Profile    := profile
        this.Lines      := []
        this.Held       := Map()
        this.UsedKeys   := Map()
        this.KeyCounts  := Map()
        this.ComboCounts := Map()
        this.LastTick   := this.StartTick := A_TickCount
        this.StartStamp := A_Now
    }

    Add(txt) {
        now := A_TickCount
        if (this.Lines.Length && now - this.LastTick > 0)
            this.Lines.Push("Sleep " (now - this.LastTick))
        this.Lines.Push(txt)
        this.LastTick := now
    }

    KeyPress(k) {
        if (this.Held.Has(k) || !WinActive(this.Win))
            return                                   ; ignore auto-repeat & other windows
        this.Held[k] := true, this.UsedKeys[k] := true
        this.KeyCounts[k] := this.KeyCounts.Get(k, 0) + 1                    ; statistics
        if !IsModifier(k) && (mods := this.Mods()) != ""
            this.ComboCounts[mods k] := this.ComboCounts.Get(mods k, 0) + 1  ; e.g. "Ctrl+C"
        this.Add('Send "{Blind}{' EscKey(k) ' down}"')
    }

    KeyRelease(k) {
        if !this.Held.Has(k)
            return
        this.Held.Delete(k)
        this.Add('Send "{Blind}{' EscKey(k) ' up}"')
    }

    MouseBtn(btn, dir, x, y) {
        if (dir = "Down") {
            if (this.Held.Has(btn) || !WinActive(this.Win))
                return
            this.Held[btn] := true, this.UsedKeys[btn] := true
        } else {
            if !this.Held.Has(btn)
                return
            this.Held.Delete(btn)
        }
        this.Add(Format('Click "{} {} {} {}"', x, y, ClickName(btn), dir))
    }

    Wheel(dir, x, y) {
        if WinActive(this.Win)
            this.Add(Format('Click "{} {} {}"', x, y, dir))
    }

    ; release anything still held when the recording stops
    ReleaseAll() {
        for k in this.Held.Clone()
            this.Add(InStr(k, "Button") ? Format('Click "{}"', ClickName(k) " Up")
                                        : 'Send "{Blind}{' EscKey(k) ' up}"')
        this.Held.Clear()
    }

    Mods() {
        p := ""
        if (this.Held.Has("LControl") || this.Held.Has("RControl"))
            p .= "Ctrl+"
        if (this.Held.Has("LAlt") || this.Held.Has("RAlt"))
            p .= "Alt+"
        if (this.Held.Has("LShift") || this.Held.Has("RShift"))
            p .= "Shift+"
        if (this.Held.Has("LWin") || this.Held.Has("RWin"))
            p .= "Win+"
        return p
    }

    Stats() {
        total := 0
        for k, n in this.KeyCounts
            total += n
        secs   := (A_TickCount - this.StartTick) // 1000
        dur    := Format("{}:{:02}:{:02}", secs // 3600, Mod(secs // 60, 60), Mod(secs, 60))
        perMin := secs ? total * 60 / secs : 0
        s := "`n; ===================== Statistics =====================`n"
        s .= "; Profile: " (IsObject(this.Profile) ? Opt(this.Profile, "Name") : "manual recording") "`n"
        s .= "; Window:  " this.Title "   (" this.Exe ")`n"
        s .= "; Started: " FormatTime(this.StartStamp, "yyyy-MM-dd HH:mm") "`n"
        s .= Format("; Duration: {}   Key presses: {}   ({:.1f} per minute)`n", dur, total, perMin)
        s .= "; (mouse clicks are not counted)`n"
        s .= "`n; ---- Keys (most used first) ----`n" CountTable(this.KeyCounts, total)
        if this.ComboCounts.Count
            s .= "`n; ---- Combinations with Ctrl / Alt / Shift / Win ----`n" CountTable(this.ComboCounts, total)
        return s
    }

    ; builds the full pattern script as text
    ; NEW: patterns have no tray icon, only one runs at a time, F10 closes a waiting pattern
    Text() {
        keys := ""
        for k in this.UsedKeys
            keys .= (keys = "" ? "" : ", ") '"' EscKey(k) '"'
        body := ""
        for l in this.Lines
            body .= "    " l "`n"

        tpl := "
        (
#Requires AutoHotkey v2.0
#SingleInstance Force
#NoTrayIcon
InstallKeybdHook
InstallMouseHook
SendMode "Event"
SetKeyDelay -1, -1
SetMouseDelay -1
SetDefaultMouseSpeed 0
CoordMode "Mouse", "Client"

; only ONE recorded pattern runs at a time -> close any other one
DetectHiddenWindows true
for h in WinGetList("RecorderPattern ahk_class AutoHotkey")
    if (h != A_ScriptHwnd)
        WinClose h
WinSetTitle "RecorderPattern", A_ScriptHwnd
DetectHiddenWindows false

; ===================== Settings =====================
GameWin  := "{{WIN}}"   ; pattern ONLY runs while this window is active
Repeat   := 1           ; how often to run the pattern (0 = forever)
UsedKeys := [{{KEYS}}]  ; released automatically on stop/abort
Playing  := false

#HotIf WinActive(GameWin)
F8::Play()              ; start pattern (only works inside the game)
#HotIf
F10::Abort()            ; while playing: stop  |  while waiting: close this pattern

OnExit((*) => ReleaseAll())
ToolTip "Pattern ready: switch to the game and press F8   (F10 = close)"
SetTimer () => ToolTip(), -4000

Play() {
    global Playing
    if Playing
        return
    Playing := true
    SetTimer Watchdog, 50
    Loop {
        Pattern()
        if (Repeat && A_Index >= Repeat)
            break
    }
    SetTimer Watchdog, 0
    ReleaseAll()
    Playing := false
}

Watchdog() {
    if !WinActive(GameWin)      ; left the game -> stop immediately
        Abort()
}

Abort(*) {
    ReleaseAll()
    if Playing {
        Reload                  ; stop playing, stay ready for F8
        return
    }
    ExitApp                     ; not playing -> close the pattern
}

ReleaseAll() {
    for k in UsedKeys
        if GetKeyState(k)
            Send "{Blind}{" k " up}"
}

; ===================== Recorded pattern =====================
; recorded {{DATE}} - edit freely
Pattern() {
{{BODY}}}
{{STATS}}
        )"
        txt := StrReplace(tpl, "{{WIN}}", "ahk_exe " this.Exe " ahk_class " this.WinClass)
        txt := StrReplace(txt, "{{KEYS}}", keys)
        txt := StrReplace(txt, "{{DATE}}", FormatTime(, "yyyy-MM-dd HH:mm"))
        txt := StrReplace(txt, "{{BODY}}", body)
        txt := StrReplace(txt, "{{STATS}}", this.Stats())
        return txt
    }

    Write(path) {
        txt := this.Text()
        try FileDelete path
        FileAppend txt, path, "UTF-8"
    }
}

; list of recordings that are running right now
Sessions() {
    list := []
    if IsObject(ManualRec)
        list.Push(ManualRec)
    if IsObject(AutoRec)
        list.Push(AutoRec)
    return list
}

; hooks are on while at least one recording runs
UpdateHooks() {
    static on := false
    want := IsObject(ManualRec) || IsObject(AutoRec)
    if (want = on)
        return
    on := want
    if want
        IH.Start()
    else
        IH.Stop()
    for h in MouseHotkeys
        Hotkey h, want ? "On" : "Off"
}

; red = manual recording, orange = only auto session, normal = nothing recording
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := IsObject(ManualRec) ? "FF9999" : IsObject(AutoRec) ? "FFCC66" : ""
}

; ---------------- Show / hide the toolbar ----------------              ; NEW (whole section)
ToolbarVisible() => DllCall("IsWindowVisible", "Ptr", G.Hwnd)

; X button: hide (recordings keep running). Shift + X = close the recorder.
HideToolbar(*) {
    if GetKeyState("Shift") {
        ExitApp                                  ; OnExit still saves / sends
    }
    G.Hide()
    ToolTip "Recorder hidden - Ctrl+F12 shows it again"
    SetTimer () => ToolTip(), -3000
}

ToggleToolbar() {
    if ToolbarVisible()
        HideToolbar()
    else
        G.Show("NoActivate")                     ; same spot, doesn't take focus from the game
}

; F9: start/stop manual recording - beeps when you can't see the toolbar
ToggleManual() {
    if (IsObject(ManualRec) || Arming) {
        StopRec()
        HiddenBeep(500)                          ; low beep = stopped
    } else {
        StartRec()
        if Arming
            HiddenBeep(1000)                     ; high beep = started
    }
}

HiddenBeep(freq) {
    if (BeepWhenHidden && !ToolbarVisible())
        SoundBeep freq, 80
}

; ---------------- Manual recording ----------------
StartRec() {
    global Arming
    if (IsObject(ManualRec) || Arming || Mailing)
        return
    Arming := true
    UpdateToolbar()
    SetTimer ArmCheck, 100
}

ArmCheck() {
    global ManualRec, Arming
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd)
        return
    try cls := WinGetClass(hwnd)
    catch
        return
    if (cls ~= "^(Progman|WorkerW|Shell_TrayWnd|Shell_SecondaryTrayWnd)$")
        return                                   ; desktop / taskbar -> keep waiting
    try s := RecSession(hwnd)
    catch
        return                                   ; window vanished -> try again
    SetTimer ArmCheck, 0
    Arming := false
    ManualRec := s
    UpdateHooks(), UpdateToolbar()
}

StopRec() {
    global Arming, ManualRec
    if Arming {
        SetTimer ArmCheck, 0
        Arming := false
    }
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0           ; auto session (if any) keeps running
        s.ReleaseAll()
        UpdateHooks()
        s.Write(LogFile)
    }
    UpdateToolbar()
}

; ---------------- Automatic game sessions ----------------
StartAuto(hwnd, profile) {
    global AutoRec
    try AutoRec := RecSession(hwnd, profile)
    catch
        return
    UpdateHooks(), UpdateToolbar()
}

; end the auto session + send it (runs when the game window closes)
StopAuto() {
    global AutoRec, SkipHwnd
    if !IsObject(AutoRec)
        return
    s := AutoRec, AutoRec := 0                   ; manual recording (if any) keeps running
    s.ReleaseAll()
    UpdateHooks(), UpdateToolbar()
    SkipHwnd := s.Hwnd                           ; don't auto-restart in THIS game window until it's closed
    QueueAuto(s)
    FlushPending()
}

; freezes a finished session (name + text) so it can wait for sending
QueueAuto(s) {
    PendingSends.Push({Name: AutoName(s), Text: s.Text(), Profile: s.Profile})
}

; send finished sessions (waits if another email is being sent right now)
FlushPending() {
    while (PendingSends.Length && !Mailing)
        SendAuto(PendingSends.RemoveAt(1))
}

AutoWatch() {
    global SkipHwnd
    FlushPending()
    if (IsObject(ManualRec) && !WinExist(ManualRec.Win))
        StopRec()                                ; manual target window closed -> save it
    if IsObject(AutoRec) {
        if !WinExist(AutoRec.Win)
            StopAuto()                           ; game window closed -> send
        return
    }
    if !AutoEnabled
        return
    if (SkipHwnd && !WinExist("ahk_id " SkipHwnd))
        SkipHwnd := 0                            ; skipped game closed -> auto-start again next time
    if Mailing
        return
    hwnd := WinExist("A")
    if (!hwnd || hwnd = G.Hwnd || hwnd = SkipHwnd)
        return
    if (p := MatchProfile(hwnd))
        StartAuto(hwnd, p)                       ; starts even while a manual recording runs
}

MatchProfile(hwnd) {
    try {
        exe   := WinGetProcessName(hwnd)
        title := WinGetTitle(hwnd)
    } catch
        return 0
    for p in AutoProfiles {
        if (Opt(p, "Exe") != "" && exe != Opt(p, "Exe"))
            continue
        if (Opt(p, "Title") != "" && !RegExMatch(title, "i)" Opt(p, "Title")))
            continue
        return p
    }
    return 0
}

Opt(o, key) => (IsObject(o) && o.HasOwnProp(key)) ? o.%key% : ""

SafeName(s) {
    s := RegExReplace(Trim(s), '[\\/:*?"<>|,`'_ ]+', "-")
    return s != "" ? s : "Auto"
}

AutoRecipient() => Trim(AutoEmailTo) != "" ? AutoEmailTo : EmailTo

RecipientFor(profile) {
    to := Trim(Opt(profile, "EmailTo"))
    return to != "" ? to : AutoRecipient()
}

; e.g. Modded_factorio_2026-11-18_20.15.30  (attachment name + email subject)
AutoName(s) {
    game := RegExReplace(s.Exe, "i)\.exe$")
    return SafeName(Opt(s.Profile, "Name")) "_" game "_" FormatTime(, "yyyy-MM-dd_HH.mm.ss")
}

; closing the recorder: save the manual recording, send the auto session
ExitSave(*) {
    global ManualRec, AutoRec
    if IsObject(ManualRec) {
        s := ManualRec, ManualRec := 0
        s.ReleaseAll()
        s.Write(LogFile)
    }
    if IsObject(AutoRec) {
        s := AutoRec, AutoRec := 0
        s.ReleaseAll()
        QueueAuto(s)
    }
    FlushPending()                               ; send whatever is still waiting before closing
}

; ---------------- Toolbar focus handling ----------------
ToolbarClick(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd)
        return PostMessage(0xA1, 2, , , G)       ; background -> drag the toolbar
    if (hwnd = RecvEdit.Hwnd)
        FocusReceiver()
}

FocusReceiver() {
    global PrevActive
    a := WinExist("A")
    if (a && a != G.Hwnd)
        PrevActive := a                          ; remember the game / previous window
    WinSetExStyle "-0x08000000", "ahk_id " G.Hwnd   ; allow the toolbar to take keyboard focus
    WinActivate "ahk_id " G.Hwnd
    RecvEdit.Focus()
}

ToolbarActivate(wp, lp, msg, hwnd) {
    if (hwnd = G.Hwnd && (wp & 0xFFFF) = 0)      ; toolbar lost focus
        WinSetExStyle "+0x08000000", "ahk_id " G.Hwnd   ; -> back to "never steal focus"
}

ReturnFocus() {
    if (PrevActive && WinExist("ahk_id " PrevActive))
        WinActivate "ahk_id " PrevActive
}

; reads the receiver box: "" = empty, otherwise "a@b.com,c@d.com"; bad = first invalid address
ParseReceivers(s, &bad := "") {
    out := "", bad := ""
    for a in StrSplit(s, [",", ";", " "]) {
        if ((a := Trim(a)) = "")
            continue
        if !(a ~= "^[^@\s]+@[^@\s]+\.[^@\s]+$") {
            bad := a
            return ""
        }
        out .= (out = "" ? "" : ",") a
    }
    return out
}

EditFile() {
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    Run 'notepad.exe "' LogFile '"'
}

; ---------------- Save to SavedPatterns ----------------
SavePattern() {
    if (IsObject(ManualRec) || Arming)
        return
    if !FileExist(LogFile)
        return MsgBox("Nothing recorded yet.", "Recorder", "Icon! T3")
    r := InputBox("Name for this pattern (saved in SavedPatterns):", "Save pattern", "w320 h120")
    if (r.Result != "OK" || Trim(r.Value) = "")
        return
    name := RegExReplace(Trim(r.Value), '[\\/:*?"<>|,`']', "_")   ; no illegal / mail-unsafe chars
    if !(name ~= "i)\.ahk$")
        name .= ".ahk"
    dest := SaveDir "\" name
    if FileExist(dest) && MsgBox('"' name '" already exists. Overwrite?', "Save pattern", "YesNo Icon?") = "No"
        return
    FileCopy LogFile, dest, 1
}

; ---------------- Send menu ----------------
SendMenu() {
    if (IsObject(ManualRec) || Arming || Mailing)
        return

    ; receiver box overrides the receivers from SETTINGS
    override := ParseReceivers(RecvEdit.Value, &bad)
    if (bad != "")
        return MsgBox('"' bad '" is not a valid email address.`nFix it in the receiver box (or clear it with Esc).', "Recorder", "Icon!")
    to := override != "" ? override : EmailTo

    m := Menu()
    if FileExist(LogFile)
        m.Add("Current recording (~pattern.ahk)", SendTo.Bind(LogFile, to))
    count := 0
    Loop Files SaveDir "\*.ahk" {
        if (count = 0 && FileExist(LogFile))
            m.Add()                              ; separator
        m.Add(A_LoopFileName, SendTo.Bind(A_LoopFileFullPath, to))
        count++
    }
    if (count = 0 && !FileExist(LogFile))
        m.Add("(nothing to send)", (*) => 0), m.Disable("(nothing to send)")

    m.Add()
    m.Add("Open SavedPatterns folder", (*) => Run(SaveDir))
    m.Show()
}

; shared patterns: regular account, real names
SendTo(path, to, *) {
    SplitPath path, &name
    SendMail(path, to
        , "AHK pattern " name " (" FormatTime(, "yyyy-MM-dd HH.mm.ss") ")"
        , "AutoHotkey pattern " name " sent from " A_ComputerName
        , ShareAccount, AliasesOnShared)
}

; auto sessions: written to a temp file, sent, then the temp file is deleted
SendAuto(item) {
    parts := StrSplit(item.Name, "_", , 2)       ; "Modded" | "factorio_2026-11-18_20.15.30"
    pname := parts[1]
    rest  := parts.Length > 1 ? StrReplace(parts[2], "_", " ") : ""
    try {
        try FileDelete AutoTmp
        FileAppend item.Text, AutoTmp, "UTF-8"
    } catch as e {
        MsgBox "Could not write the auto session temp file:`n" e.Message, "Recorder", "Icon!"
        return
    }
    SendMail(AutoTmp, RecipientFor(item.Profile)
        , "[" pname "] Session " rest, BodyFromText(item.Text)
        , SessionAccount, AliasesOnSessions, item.Name ".ahk")
    try FileDelete AutoTmp
}

; takes the statistics block from the end of a pattern as email text
BodyFromText(txt) {
    pos := InStr(txt, "; ===================== Statistics", , -1)
    if !pos
        return "AutoHotkey session sent from " A_ComputerName
    out := ""
    Loop Parse SubStr(txt, pos), "`n", "`r"
        out .= RegExReplace(A_LoopField, "^;\s?") "`n"
    return "Sent from " A_ComputerName "`n" out
}

; fallback when smtp can't be used: Thunderbird first, then classic Outlook
PickMailProgram() {
    if TBPath()
        return "thunderbird"
    if OutlookInstalled()
        return "outlook"
    return ""
}

OutlookInstalled() {
    try {
        RegRead("HKCR\Outlook.Application\CLSID")   ; only classic Outlook registers this
        return true
    }
    return false
}

; attachName = name the attachment gets in the email ("" = same as the file)
SendMail(path, to, subject, body, acc, useAliases, attachName := "") {
    global Mailing
    if Mailing
        return false
    to := Trim(to)
    sendNow := (to != "")                        ; no receiver -> just open the email to fill in

    ; smtp can't be used without a login, and can't open an email for editing
    ; -> use Thunderbird, or else Outlook, as if Method were set to that
    method := Opt(acc, "Method")
    if (method = "smtp" && (!sendNow || !SmtpConfigured(acc))) {
        method := PickMailProgram()
        if (method = "") {
            which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
            MsgBox "Nothing was sent: no email program found.`n`n"
                 . (sendNow ? which " has no login in SETTINGS, and neither Thunderbird nor classic Outlook is installed."
                            : "No receiver was set, and opening an email to fill in needs Thunderbird or classic Outlook.")
                 , "Recorder", "Icon!"
            return false
        }
    }

    Mailing := true
    SplitPath path, &name
    if (attachName != "")
        name := attachName

    ; privacy - only when this kind of mail is set to use aliases
    if useAliases {
        subject := Anonymize(subject)
        body    := Anonymize(body)
    }
    attach := PrepareAttachment(path, name, useAliases)
    if !attach {
        Mailing := false
        return false
    }

    if (method = "smtp")
        ok := SendSmtp(acc, attach, subject, body, to)
    else if (method = "thunderbird")
        ok := SendThunderbird(attach, subject, body, to, sendNow)
    else if (method = "outlook")
        ok := SendOutlook(attach, subject, body, to, sendNow)
    else {
        MsgBox 'Unknown Method "' method '" in the account settings.`nUse "smtp", "thunderbird" or "outlook".', "Recorder", "Icon!"
        ok := false
    }
    if sendNow                                   ; open email still needs the file -> keep it
        try FileDelete attach                    ; remove the temp copy
    Mailing := false
    return ok
}

; ===================== EMAIL PRIVACY =====================
; turns the alias text from SETTINGS into a list, longest "real" text first
ParseAliases(text) {
    list := []
    Loop Parse text, "`n", "`r" {
        line := Trim(A_LoopField)
        if (line = "" || SubStr(line, 1, 1) = ";" || !(pos := InStr(line, "=")))
            continue
        real := Trim(SubStr(line, 1, pos - 1))
        if (real != "")
            list.Push([real, Trim(SubStr(line, pos + 1))])
    }
    ; longest first, so "My Game 2" is replaced before "My Game"
    Loop list.Length - 1 {
        i := A_Index
        Loop list.Length - i {
            j := A_Index
            if StrLen(list[j][1]) < StrLen(list[j + 1][1])
                tmp := list[j], list[j] := list[j + 1], list[j + 1] := tmp
        }
    }
    return list
}

Anonymize(text) {
    for pair in EmailAliasList
        text := StrReplace(text, pair[1], pair[2], false)   ; false = ignore case
    return text
}

; makes a temp copy of the file to attach (original stays untouched)
PrepareAttachment(path, name, useAliases) {
    if !FileExist(path) {
        MsgBox "File not found:`n" path, "Recorder", "Icon!"
        return ""
    }
    safeName := RegExReplace(useAliases ? Anonymize(name) : name, '[\\/:*?"<>|]', "_")
    if AttachAsTxt
        safeName .= ".txt"
    DirCreate MailTempDir
    attach := MailTempDir "\" safeName
    try {
        txt := FileRead(path, "UTF-8")
        if (useAliases && AnonymizeAttachment)
            txt := Anonymize(txt)
        try FileDelete attach
        FileAppend txt, attach, "UTF-8"
    } catch as e {
        MsgBox "Could not prepare the attachment:`n" e.Message, "Recorder", "Icon!"
        return ""
    }
    return attach
}

; ---------------- SMTP via PowerShell ----------------
SmtpConfigured(acc) => (Trim(Opt(acc, "User")) != "" && Trim(Opt(acc, "Password")) != "")

SendSmtp(acc, attach, subject, body, to) {
    which := (acc = ShareAccount) ? "ShareAccount" : "SessionAccount"
    if !SmtpConfigured(acc) {
        MsgBox which " is not set up yet - nothing was sent.`n"
             . "Fill in User and Password for " which " in the SETTINGS section.", "Recorder", "Icon!"
        return false
    }
    if !FileExist(attach) {
        MsgBox "Attachment not found:`n" attach, "Recorder", "Icon!"
        return false
    }

    psFile  := A_Temp "\recorder_sendmail.ps1"
    errFile := A_Temp "\recorder_sendmail_err.txt"

    ps := "
    (
$ErrorActionPreference = 'Stop'
try {
    $pw   = ConvertTo-SecureString $env:REC_PASS -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential($env:REC_USER, $pw)
    $to   = @($env:REC_TO -split '\s*[,;]\s*' | Where-Object { $_ -ne '' })
    $p = @{
        SmtpServer  = $env:REC_SERVER
        Port        = [int]$env:REC_PORT
        UseSsl      = $true
        Credential  = $cred
        From        = $env:REC_USER
        To          = $to
        Subject     = $env:REC_SUBJ
        Body        = $env:REC_BODY
        Attachments = $env:REC_ATTACH
        Encoding    = [System.Text.Encoding]::UTF8
    }
    Send-MailMessage @p -WarningAction SilentlyContinue
    exit 0
} catch {
    $_.Exception.Message | Out-File -FilePath $env:REC_ERR -Encoding UTF8
    exit 1
}
    )"
    try FileDelete psFile
    try FileDelete errFile
    FileAppend ps, psFile, "UTF-8"

    vars := Map(
        "REC_SERVER", Opt(acc, "Server"),
        "REC_PORT",   Opt(acc, "Port"),
        "REC_USER",   Trim(Opt(acc, "User")),
        "REC_PASS",   StrReplace(Opt(acc, "Password"), " "),
        "REC_TO",     Trim(to),
        "REC_SUBJ",   subject,
        "REC_BODY",   body,
        "REC_ATTACH", attach,
        "REC_ERR",    errFile
    )
    for k, v in vars
        EnvSet k, v

    ; toolbar freezes for a few seconds while sending
    exitCode := RunWait('powershell.exe -NoProfile -ExecutionPolicy Bypass -File "' psFile '"', , "Hide")

    EnvSet "REC_PASS"                    ; remove the password from the environment again
    try FileDelete psFile

    if (exitCode = 0)
        return true

    err := FileExist(errFile) ? Trim(FileRead(errFile, "UTF-8")) : "Unknown error (exit code " exitCode ")"
    if (InStr(err, "5.7.8") || InStr(err, "not accepted") || InStr(err, "authenticat"))
        hint := "The mail server rejected the login of " which ".`nCheck User (full address) and the app password OF THAT account."
    else if (InStr(err, "connect") || InStr(err, "timed out") || InStr(err, "Failure sending mail"))
        hint := "Could not reach " Opt(acc, "Server") ":" Opt(acc, "Port") ".`nCheck internet / firewall / antivirus."
    else
        hint := "Check internet, app password and 2-Step Verification of " which "."
    MsgBox "Sending from " Trim(Opt(acc, "User")) " failed:`n`n" err "`n`n" hint, "Recorder", "Icon!"
    return false
}

; ---------------- Thunderbird ----------------
TBPath() {
    try {
        p := RegRead("HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\thunderbird.exe")
        if FileExist(p)
            return p
    }
    for p in [A_ProgramFiles "\Mozilla Thunderbird\thunderbird.exe"
            , "C:\Program Files (x86)\Mozilla Thunderbird\thunderbird.exe"]
        if FileExist(p)
            return p
    return ""
}

; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendThunderbird(attach, subject, body, to, sendNow := true) {
    exe := TBPath()
    if !exe {
        MsgBox "Thunderbird not found.", "Recorder", "Icon!"
        return false
    }
    to := RegExReplace(Trim(to), "\s*[,;]\s*", ",")   ; Thunderbird wants a,b
    ; command line can't take line breaks / quotes -> flatten the text
    body := StrReplace(StrReplace(RegExReplace(body, "[`r`n]+", " | "), "'", ""), '"', "")
    url  := "file:///" StrReplace(StrReplace(attach, "\", "/"), " ", "%20")
    args := Format("subject='{}',body='{}',attachment='{}'", subject, body, url)
    if (to != "")
        args := "to='" to "'," args               ; empty -> receiver field stays blank
    Run Format('"{}" -compose "{}"', exe, args)

    hwnd := 0, start := A_TickCount
    while (!hwnd && A_TickCount - start < 30000) {
        for h in WinGetList("ahk_exe thunderbird.exe") {
            try t := WinGetTitle(h)
            catch
                continue
            if InStr(t, subject) {
                hwnd := h
                break
            }
        }
        if !hwnd
            Sleep 250
    }
    if !hwnd {
        MsgBox "Thunderbird's compose window did not appear.", "Recorder", "Icon!"
        return false
    }
    Sleep 1500
    WinActivate hwnd
    if !sendNow                                  ; leave it open for you to fill in
        return true
    if !WinWaitActive(hwnd, , 3) {
        MsgBox "Couldn't focus the Thunderbird window - please press Send yourself.", "Recorder", "Icon!"
        return false
    }
    Send "^{Enter}"
    if WinWaitClose(hwnd, , 15)
        return true
    MsgBox "Thunderbird is probably asking for confirmation.`n"
         . "Click Send there and tick 'Don't ask me again' - next time it's fully automatic.", "Recorder", "Iconi"
    return false
}

; ---------------- Outlook (classic) ----------------
; sendNow = false -> only open the email (e.g. no receiver), you press Send yourself
SendOutlook(attach, subject, body, to, sendNow := true) {
    try {
        if !ProcessExist("outlook.exe") {
            Run "outlook.exe"
            WinWait "ahk_exe outlook.exe", , 60
            Sleep 5000
        }
        ol   := ComObject("Outlook.Application")
        mail := ol.CreateItem(0)
        if (Trim(to) != "")
            mail.To := RegExReplace(Trim(to), "\s*[,;]\s*", ";")   ; Outlook wants a;b
        mail.Subject := subject
        mail.Body    := body
        mail.Attachments.Add(attach)
        if sendNow
            mail.Send()
        else
            mail.Display()                       ; open it for you to fill in
        return true
    } catch as e {
        MsgBox "Outlook failed:`n" e.Message "`n`n"
             . "Note: only classic Outlook can be automated. The 'new Outlook' app doesn't support it.", "Recorder", "Icon!"
        return false
    }
}

; ---------------- Event handlers (passed on to every running recording) ----------------
KeyDown(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyPress(k)
}

KeyUp(ih, vk, sc) {
    k := KeyName(vk, sc)
    for s in Sessions()
        s.KeyRelease(k)
}

MouseEvt(btn, dir, *) {
    MouseGetPos &x, &y, &win
    if (dir = "Down" && win = G.Hwnd)
        return                                   ; clicks on the toolbar are never recorded
    for s in Sessions()
        s.MouseBtn(btn, dir, x, y)
}

WheelEvt(dir, *) {
    MouseGetPos &x, &y, &win
    if (win = G.Hwnd)
        return
    for s in Sessions()
        s.Wheel(dir, x, y)
}

; ---------------- Helpers ----------------
KeyName(vk, sc) {
    name := GetKeyName(Format("vk{:x}sc{:x}", vk, sc))
    return name != "" ? name : Format("vk{:X}sc{:03X}", vk, sc)
}

ClickName(btn) => Map("LButton", "Left", "RButton", "Right", "MButton", "Middle"
                    , "XButton1", "X1", "XButton2", "X2")[btn]

EscKey(k) => StrReplace(StrReplace(k, "``", "````"), '"', '``"')

IsModifier(k) => k ~= "i)^[LR](Control|Shift|Alt|Win)$"

CountTable(counts, total) {
    list := ""
    for k, n in counts
        list .= Format("{:09}", n) "`t" k "`n"   ; zero-padded so a text sort = number sort
    if (list = "")
        return ";   (none)`n"
    out := ""
    Loop Parse Sort(RTrim(list, "`n"), "R"), "`n" {
        parts := StrSplit(A_LoopField, "`t", , 2)
        n := Integer(parts[1])
        out .= Format(";   {:-24}{:8}   {:5.1f} %`n", parts[2], n, total ? n * 100 / total : 0)
    }
    return out
}


### 🤖 Assistant (Message 76)

Apply these edits to the supplied script. They leave **F9** as the manual start/stop key, make manual recording orange, and keep the toolbar’s **X** as a silent hide button.

Replace the toolbar settings comments and remove `BeepWhenHidden`:

```autohotkey
; ---- TOOLBAR ----
; X = hide toolbar; recordings continue in the background.
```

Update the GUI color comment and X-button handler:

```autohotkey
; colors: orange = manual recording, normal = nothing recording
```

```autohotkey
BtnExit.OnEvent("Click", HideToolbar)   ; X = hide toolbar
```

Remove this line from the keyboard-hook section:

```autohotkey
IH.KeyOpt("{F12}", "-N")
```

Replace the hotkey block with:

```autohotkey
F9::ToggleManual()                   ; manual recording start/stop
```

Remove the `^F12` and `^+F12` hotkey lines.

Replace `UpdateToolbar()` and the entire section from `; ---------------- Show / hide the toolbar ----------------` through `HiddenBeep()` with:

```autohotkey
; orange = manual recording (or waiting to start); auto-only sessions don't change the color
UpdateToolbar() {
    man := IsObject(ManualRec) || Arming
    BtnStart.Enabled := !man, BtnStop.Enabled := man
    BtnSave.Enabled  := !man, BtnSend.Enabled := !man
    G.BackColor := man ? "FFCC66" : ""
}

; X hides the toolbar; any recordings keep running.
HideToolbar(*) {
    G.Hide()
}

; F9 starts/stops manual recording without beeps.
ToggleManual() {
    if (IsObject(ManualRec) || Arming)
        StopRec()
    else
        StartRec()
}
```

Keep `#NoTrayIcon` as-is. Since the script has no tray icon and no exit hotkey after these changes, close it through Task Manager when needed.

