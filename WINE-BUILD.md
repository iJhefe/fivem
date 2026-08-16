# Build do cliente FiveM com compatibilidade Wine/Proton

Este fork tem a branch **`wine-compat`**, com três correções que permitem ao
cliente FiveM (Legacy) rodar sob Wine/Proton no Linux.

**Esta build precisa ser compilada no Windows.** É essa a razão de este documento
existir: quem vai usar o cliente está no Linux, quem compila está no Windows.

## O que o patch faz

Três lugares no código assumem o layout de memória do `ntdll` do Windows. No Wine
o layout é outro, e cada um quebra de um jeito. Todos passam a ser guardados por
`CfxIsWine()`:

| Arquivo | Problema no Wine | Sintoma |
|---|---|---|
| `launcher/UpdaterUI.cpp` | TenUI usa `WindowsXamlManager` (XAML Islands), não implementado | `c0000409` antes de baixar nada; `FiveM.app` vazia |
| `citicore/DllGameComponent.Win32.cpp` | patch de NtGlobalFlag escreve em `_fltused - 16` | `c0000005` em `CoreRT.dll+1C471` |
| `citicore/SEHTableHandler.Win32.cpp` | desmonta `RtlLookupFunctionTable` procurando chamada interna | processo do jogo morre ao iniciar |

Crédito do patch: [Gogsi](https://github.com/Gogsi), commit
[`8ef2fcc`](https://github.com/Gogsi/fivem/commit/8ef2fccd7ddabf21f459ec906363f1c05447d876).
Contexto: [PR #4050](https://github.com/citizenfx/fivem/pull/4050) e
[PR #4095](https://github.com/citizenfx/fivem/pull/4095).

O anticheat **não é tocado**. O `ComponentLoader` já troca `adhesive` por
`sticky` ao detectar Wine, ou seja, o cliente roda em **modo inseguro**. Serve
para servidor local de desenvolvimento; não serve para servidor público.

## Pré-requisitos (na máquina Windows)

Conforme [docs/building.md](docs/building.md) do projeto:

- Windows 10/11 de 64 bits
- **Visual Studio 2022** com os workloads:
  - Desenvolvimento para desktop com C++
  - Desenvolvimento para desktop .NET
  - Desenvolvimento de plataforma cruzada .NET Core
- **.NET Framework 4.6 targeting pack**
- **Windows 11 SDK 10.0.22000**
- PowerShell 7, Python 3, MSYS2, Node.js + Yarn
- ~30 GB livres (o repositório completo com submódulos é grande)

## Passos

```powershell
# 1. Clonar ESTA branch, com submódulos
git clone --recurse-submodules -b wine-compat https://github.com/iJhefe/fivem.git
cd fivem

# 2. Preparar (baixa Chrome embutido e dependências nativas)
.\prebuild.cmd

# 3. Gerar a solução do Visual Studio
.\fxd gen -game five

# 4. Compilar
.\fxd vs -game five
```

O passo 4 abre o Visual Studio. Compile em **Release**, plataforma **x64**.

## O que entregar de volta

O objetivo é substituir os binários que o updater baixou. Os artefatos ficam em
`code\bin\five\release\`. Os que importam:

- **`CoreRT.dll`** — contém `DllGameComponent` e `SEHTableHandler` (correções 2 e 3)
- **`FiveM.exe`** — contém `UpdaterUI` (correção 1)

Compacte esses dois e mande.

> Se der para compilar só o projeto **CitiCore** (que gera `CoreRT.dll`), já
> resolve as duas correções fatais — a do `UpdaterUI` tem contorno por variável
> de ambiente (`CitizenFX_NoTenUI=1`). Mas gerar a solução exige o toolchain
> completo de qualquer forma.

## Do lado Linux (depois de receber os binários)

Compilar é metade do trabalho. Esta seção documenta o resto, tudo **testado** —
com o cliente entrando num servidor local e criando personagem.

### 1. Instalar os binários

```bash
cd .../client/FiveM.app
cp CoreRT.dll CoreRT.dll.orig          # backup
cp /caminho/novo/CoreRT.dll .
```

O `FiveM.exe` também pode ser trocado, mas **o updater o reverte** (aconteceu no
primeiro lançamento). Não é problema: a correção dele é a do `UpdaterUI`, que tem
contorno pela variável `CitizenFX_NoTenUI=1`.

### 2. Runtime C++ compatível

O `CoreRT.dll` é linkado contra o runtime do Visual Studio da máquina de build,
mas o FiveM distribui o **próprio** runtime em `FiveM.app/bin/`, de outra safra.
Sem casar os dois, o cliente crasha em `MSVCP140.dll+13028`.

```bash
protontricks --no-bwrap <APPID> -q vcrun2022
# copiar de pfx/drive_c/windows/system32 para FiveM.app/bin/:
#   msvcp140.dll msvcp140_1.dll msvcp140_2.dll msvcp140_atomic_wait.dll
#   msvcp140_codecvt_ids.dll vcruntime140.dll vcruntime140_1.dll
```

> O `winetricks` do Ubuntu 24.04 (9.0) não reconhece o `wineserver` do GE-Proton
> 11 e falha em silêncio. Use o upstream:
> `curl -L -o ~/.local/bin/winetricks https://raw.githubusercontent.com/Winetricks/winetricks/master/src/winetricks`
>
> E o `protontricks` precisa de `--no-bwrap` no Ubuntu 24.04, porque
> `kernel.apparmor_restrict_unprivileged_userns=1` bloqueia user namespaces fora
> do perfil AppArmor do Steam.

### 3. Prefixo

- **Prefixo próprio do FiveM**, não o do GTA V: o Proton reescreve a versão do
  Windows do prefixo do jogo a cada lançamento
- **Rockstar Games Launcher instalado dentro dele** — o FiveM procura por
  `C:\Program Files\Rockstar Games\Launcher\Launcher.exe`
- **Logar no launcher ao menos uma vez, com o FiveM fechado** (ele é de instância
  única). É o login que grava `AppData/Local/DigitalEntitlements`, sem o qual o
  servidor recusa a conexão
- **Registro apontando o jogo da Steam**, senão o launcher tenta instalar um GTA V
  que já existe e trava em "Updating":

```
[HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Rockstar Games\Grand Theft Auto V]
"InstallFolderSteam"="Z:\\caminho\\para\\steamapps\\common\\Grand Theft Auto V\\"
```

### 4. O servidor também precisa de ajuste

O cliente sob Wine é **sempre inseguro** (o `ComponentLoader` troca `adhesive`
por `sticky`) e nunca emite ticket. Só entra em servidor preparado:

| Ajuste | Sem ele |
|---|---|
| `+set sv_lan 1` na linha de comando | `No authentication ticket was specified` |
| remover `"svadhesive"` de `components.json` | `Could not get resource mounter for resource X` |

E se o servidor usa **Qbox/qbx_core** (ou fork), há um terceiro: o modo LAN
desliga os identificadores, o jogador chega sem `license:` e o framework recusa
com `No Valid Rockstar License Found`. É preciso um identificador substituto em
três pontos — no `playerConnecting`, no `Login` e no `CheckPlayerData` —, e ele
deve derivar do **nome**, nunca do `source` (que muda entre a conexão e o login).

> Nada disso serve para servidor público: é configuração de desenvolvimento
> local, e desliga a validação de posse do jogo.

## Ambiente validado

Ubuntu 24.04.4 (kernel 7.0), RTX 3060 (driver 595.84), Steam nativo,
GE-Proton11-5, GTA V **Legacy** (appid 271590) build 3258, FXServer 25770.
