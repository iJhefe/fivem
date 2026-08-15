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

```bash
# a instalação do cliente fica aqui
cd /mnt/workspace/pessoal/fivem/client/FiveM.app

# backup antes de trocar
cp CoreRT.dll CoreRT.dll.orig

# substituir pelos compilados
cp /caminho/dos/novos/CoreRT.dll .
```

**Atenção:** o updater do FiveM pode sobrescrever os binários customizados na
próxima atualização. Se o crash voltar do nada, é o primeiro suspeito — basta
recopiar.

Ambiente validado do lado Linux (ver `docs/cliente-linux.md` no projeto do
servidor):

- Ubuntu 24.04, GE-Proton11-5, prefixo com `win11` (build 22000)
- Rockstar Games Launcher instalado dentro do prefixo
- GTA V **Legacy** (appid 271590), build 3258
