# peerpad.nvim

독립 Neovim 프로세스 사이의 실시간 공동 편집 플러그인입니다.
다른 플러그인이나 외부 서버 실행 파일이 필요하지 않습니다.
NFS로 같은 파일을 보는 서로 다른 노드에서도 사용할 수 있습니다.
편집 내용과 참가자 커서는 TCP로 전송하므로 노드 사이의 포트 연결이 필요합니다.

## 예시: 서로 다른 서버에서 함께 디버깅

두 사람이 서로 다른 서버에서 같은 NFS 파일을 엽니다. 한 사람이 `:Peerpad`를
실행하고 다른 사람이 참여 안내를 수락하면, 저장 전 편집과 커서를 함께 볼 수 있습니다.
완성된 내용은 주최자가 원본 파일에 저장합니다.

## 데모

[![키 입력과 양쪽 편집 반영을 보여주는 Peerpad 데모](assets/peerpad-demo.gif)](https://github.com/Sunwook-Hwang/peerpad.nvim/raw/refs/heads/main/assets/peerpad-demo.mp4)

공유 시작 → 참여 → 양쪽 편집 → 주최자 저장 → 연결 종료를 보여줍니다.
화면 하단에 사용한 키가 표시되며, 데모에서는 Space를 리더 키로 사용합니다.
[MP4 보기·다운로드](https://github.com/Sunwook-Hwang/peerpad.nvim/raw/refs/heads/main/assets/peerpad-demo.mp4).

macOS에서 독립적인 Neovim 프로세스 두 개를 로컬 TCP로 연결하고,
실제 Neovim 화면 출력을 기록했습니다. 별도 서버나 NFS 환경을 촬영한 영상은 아닙니다.
단계 설명과 키 입력 표시를 추가했으며 접속 토큰은 숨겼습니다.

## 설치

공개 저장소에서 설치할 수 있습니다.
설치 방법 하나만 선택하세요. `<leader>`는 사용자가 설정한 리더 키를 뜻합니다.
`vim.g.mapleader`는 플러그인 로드 전에 지정합니다. Neovim 0.12 이상이 필요합니다.

### 내장 `vim.pack`

`init.lua`에 추가합니다.

```lua
vim.pack.add({
    { src = "https://github.com/Sunwook-Hwang/peerpad.nvim" },
})
require("peerpad").setup({ keymaps = true })
```

### lazy.nvim

플러그인 목록에 추가합니다.

```lua
{
    "Sunwook-Hwang/peerpad.nvim",
    lazy = false, -- 파일 읽기 전에 자동 참가 안내를 등록합니다.
    main = "peerpad",
    opts = { keymaps = true },
}
```

시작할 때는 작은 설정 모듈만 로드합니다. 연결·편집 연산 코드는 공유 시작·참여
또는 안내 파일이 있는 세션을 발견할 때 로드합니다.
명령 실행 시에만 플러그인을 로드하면 그 전에 연 파일의 자동 참가 안내를 놓칠 수 있습니다.

### packer.nvim

기존 `require("packer").startup(function(use)` 안에 추가합니다.

```lua
use({
    "Sunwook-Hwang/peerpad.nvim",
    config = function()
        require("peerpad").setup({ keymaps = true })
    end,
})
```

`:PackerSync`로 설치합니다. Packer는 유지보수가 중단되어 기존 사용자를 위한 예시입니다.

### vim-plug

`plug#begin()` / `plug#end()` 사이에 추가합니다.

```vim
Plug 'Sunwook-Hwang/peerpad.nvim'
```

`call plug#end()` 뒤에 추가합니다.

```vim
lua require('peerpad').setup({ keymaps = true })
```

`:PlugInstall`로 설치합니다.

### 로컬 복사 / 인터넷이 없는 환경

패키지 폴더 전체를 복사하고 `init.lua`에 절대 경로를 추가합니다.

```lua
vim.opt.runtimepath:prepend(vim.fn.expand("~/src/peerpad.nvim"))
require("peerpad").setup({
    discovery = true,
    max_peers = 8, -- 주최자 포함, 2~64명
    keymaps = true,
})
```

경로는 실제 복사한 위치로 바꾸세요. FLASH·dotfiles·LSP·Treesitter에 의존하지 않습니다.
네이티브 `pack/*/start` 플러그인으로 설치하면 명령어도 자동 등록됩니다.
자동 발견은 파일을 읽을 때 한 번만 검사합니다.

예시는 공식 [vim.pack](https://neovim.io/doc/user/pack.html),
[lazy.nvim](https://lazy.folke.io/spec), [packer.nvim](https://github.com/wbthomason/packer.nvim),
[vim-plug](https://github.com/junegunn/vim-plug) 설치 방식에 맞췄습니다.

## 단축키

`setup({ keymaps = true })`로 일반 모드 단축키를 등록합니다.
기본값은 꺼짐이며 이미 등록된 키는 덮어쓰지 않습니다.
`P`는 대문자이므로 Shift+p입니다. `<leader>Ps`는 리더 키 → Shift+p → s입니다.
각 키에 설명도 등록하므로 키맵 목록이나 설치된 which-key에서 확인할 수 있습니다.
which-key 자체는 필요하지 않습니다.

| 키 | 명령 | 동작 |
| --- | --- | --- |
| `<leader>Ps` | `:Peerpad` | 현재 소스 공유 시작 (start) |
| `<leader>Pj` | `:PeerpadJoin` | 현재 파일의 안내된 세션 참여 (join) |
| `<leader>Pq` | `:PeerpadStop` | 연결 종료 / 주최 서버 종료 (quit) |
| `<leader>Pi` | `:PeerpadStatus` | 세션 상태 확인 (info) |

주소·포트·토큰을 직접 지정할 때는 `:PeerpadJoin 주소 포트 토큰`을 사용합니다.
자동 등록 대신 직접 매핑하려면 설치 예시의 `keymaps = true`를 아래 설정으로
바꾸세요. 다음은 같은 기본 키를 수동 등록하는 예시이며, 왼쪽 키를 원하는 키로
바꿀 수 있습니다. 자동 등록과 수동 등록 중 하나만 사용하세요.

```lua
require("peerpad").setup({ keymaps = false })

vim.keymap.set("n", "<leader>Ps", "<Cmd>Peerpad<CR>", {
    desc = "Peerpad: start sharing",
})
vim.keymap.set("n", "<leader>Pj", "<Cmd>PeerpadJoin<CR>", {
    desc = "Peerpad: join current file",
})
vim.keymap.set("n", "<leader>Pq", "<Cmd>PeerpadStop<CR>", {
    desc = "Peerpad: disconnect",
})
vim.keymap.set("n", "<leader>Pi", "<Cmd>PeerpadStatus<CR>", {
    desc = "Peerpad: session information",
})
```

## 사용

주최자가 이름 있는 UTF-8 파일을 저장한 뒤 `:Peerpad`를 실행합니다.
다른 참가자는 같은 파일을 편집창에서 열 때 참여 안내를 받습니다.
이미 열어 두었다면 `:PeerpadJoin`으로 참여합니다.
공유 저장소 없이 직접 연결하려면 주최자의 `:messages`에 나온 명령을 사용합니다.

```vim
:PeerpadJoin <주최자주소> <포트> <토큰>
```

| 명령 / 키 | 동작 |
| --- | --- |
| `:Peerpad [포트] [바인드주소]` | 현재 소스 공유; 기본은 `0.0.0.0`의 자동 선택 포트 |
| `:PeerpadJoin [주소 포트 토큰]` | 현재 파일의 세션 또는 지정 세션 참여 |
| `:PeerpadStatus` | 역할·변경 번호·대기 편집·참가자 커서 확인 |
| `:PeerpadStop` | 연결 종료; 주최자는 서버도 종료 |
| `u` / `Ctrl+r` | 자신의 공유 편집만 취소 / 다시 실행 |
| `:w` | 주최자만 원본 버퍼를 통해 동기화된 내용 저장 |

공유를 시작하면 새 공유 버퍼에서 편집해야 합니다.
원본 버퍼나 디스크가 별도로 변경되면 저장을 막아 보호합니다.
연결이 끝나면 알림을 표시합니다. 공유 내용이 열린 원본과 같고 대기 편집이 없으면
창 분할을 유지한 채 원본으로 돌아가고 중복 공유 버퍼를 제거합니다.
내용이 다르거나 원본을 확인할 수 없으면 `[disconnected]` 복구 버퍼를 남깁니다.
복구 내용은 일반 버퍼로 복사해 저장하세요. 종료 시 원본을 덮어쓰거나 다시 읽지 않습니다.
한 Neovim 프로세스에서는 세션 하나만 사용할 수 있습니다.

## Neovim 0.13 호환

Neovim 0.13에서는 `:restart`를 포함한 네이티브 세션 저장에서 공유 버퍼와
연결 종료 후 남은 스냅샷을 제외합니다. TCP 공동 편집 세션은 `peerpad://` 파일
이름으로 복원할 수 없습니다. 재시작 전에 주최자는 동기화된 내용을 `:w`로
저장하고, 참가자는 보관할 스냅샷을 일반 파일에 복사하세요. 재시작 후 다시 연결합니다.

필터는 세션 저장 때만 실행되고 저장 후 버퍼와 화면을 복원합니다. 공유 창은 세션에
원본 버퍼 화면으로 기록하며, 원본이 제거되었으면 빈 화면으로 기록합니다.
네이티브 autoread, `Q`, `.` 동작은 변경하지 않습니다. Neovim 0.12 동작은 유지합니다.

## 알아둘 점

- 연결은 암호화되지 않습니다. 신뢰할 수 있는 네트워크나 SSH 터널을 사용하세요.
  토큰을 아는 사람은 참여할 수 있으므로 공개하지 마세요.
- FLASH에 포함된 Peerpad 구현과 호환되도록 `.<파일명>.flash-share` 안내 파일과 통신 규약을 유지합니다.
- Git 프로젝트에서는 안내 파일과 임시 파일을 로컬 `info/exclude`에 제외합니다.
  안전하게 제외하거나 파일을 생성하지 못해도 수동 연결은 가능합니다.
  Git은 이 제외 처리에만 사용하며 편집 전송에는 필요하지 않습니다.
- 원본 쓰기 권한의 소유자·그룹·기타 구분에 따라 안내 파일의 읽기 권한을 설정합니다.
  ACL이나 네트워크 접근 제어를 대신하지는 않습니다.
- 백그라운드 프리뷰에는 참여 안내를 띄우지 않습니다.
  연결 중 대상 창이나 내용이 바뀌면 덮어쓰지 않고 `:buffer`로 공유 버퍼를 열 수 있게 합니다.
- UTF-8 텍스트를 대상으로 하며 문서는 1 MiB까지, 마지막 개행은 유지합니다.
  대기 편집·undo 기록·뒤처진 변경 이력에도 상한이 있습니다.
- 공유 버퍼는 별도 `acwrite` 버퍼입니다. LSP·포매터 지원은 이 플러그인이 제공하지 않습니다.
- 정상 종료 시 안내 파일을 지웁니다. 다른 호스트에서 비정상 종료한 경우 수동 정리가 필요할 수 있습니다.

참가자 색상은 현재 테마의 진단·Visual 그룹을 따릅니다.
`LiveSharePeer1`~`LiveSharePeer6`으로 변경할 수 있으며 아이콘 폰트는 필요하지 않습니다.

[English guide](README.md)
