---
title: 自动化Nuget发布
date: 2026-03-27 19:19:31
tags:
- 笔记
categories:
- [笔记,.NET]
---

# 使用 MinVer + NuGet + GitHub Actions 实现 .NET 库自动化发布

## 前言

利用 **MinVer**、**NuGet** 和 **GitHub Actions** 实现一套完整的自动化发布流程

只需打一个 Git Tag 自动完成版本计算、打包和发布到 NuGet。

## 整体架构

``` 
Git Tag (v1.0.0) 
    ↓
GitHub Actions 触发
    ↓
MinVer 从 Git 历史计算版本号
    ↓
dotnet pack 生成 NuGet 包
    ↓
自动发布到 NuGet.org
``` 

## 一、MinVer：基于 Git Tag 的版本管理

### 什么是 MinVer？

MinVer 是一个极简的 .NET 版本管理工具，它通过分析 Git 标签和提交历史来自动计算版本号，无需额外的配置文件。

### MinVer 版本计算规则

MinVer 遵循语义化版本（SemVer）规范：

| 场景 | 版本示例 | 说明 |
|------|----------|------|
| 有标签 `v1.0.0` | `1.0.0` | 直接使用标签版本 |
| 标签后有 1 个提交 | `1.0.1-alpha.1` | 自动递增 patch，添加预发布标识 |
| 标签后有 5 个提交 | `1.0.1-alpha.5` | 提交数作为预发布号 |
| 标签后无新提交 | `1.0.0` | 使用标签版本 |
| 无标签 | `0.0.0-alpha.1` | 从 0.0.0 开始 |

### 在项目中配置 MinVer

#### 1. 安装 MinVer NuGet 包

在 `.csproj` 文件中添加：

``` xml
<ItemGroup>
  <PackageReference Include="MinVer" Version="8.0.0-alpha.1">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
``` 

#### 2. 配置 MinVer 属性

``` xml
<PropertyGroup>
  <!-- 标签前缀，用于识别版本标签 -->
  <MinVerTagPrefix>v</MinVerTagPrefix>
  
  <!-- 自动递增级别：major, minor, patch -->
  <MinVerAutoIncrement>patch</MinVerAutoIncrement>
  
  <!-- 可选：指定最低版本 -->
  <!-- <MinVerMinimumMajorMinor>1.0</MinVerMinimumMajorMinor> -->
  
  <!-- 可选：指定默认预发布版本 -->
  <!-- <MinVerDefaultPreReleasePhase>alpha</MinVerDefaultPreReleasePhase> -->
</PropertyGroup>
``` 

#### 3. MinVer 配置详解

| 属性 | 说明 | 默认值 |
|------|------|--------|
| `MinVerTagPrefix` | Git 标签前缀 | 空 |
| `MinVerAutoIncrement` | 自动递增级别 | `patch` |
| `MinVerMinimumMajorMinor` | 最低主版本号 | 无 |
| `MinVerDefaultPreReleasePhase` | 默认预发布阶段 | `alpha` |
| `MinVerVerbosity` | 日志详细程度 | `normal` |

## 二、NuGet 包配置

### 完整的 NuGet 包元数据配置

``` xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    
    <!-- MinVer 配置 -->
    <MinVerTagPrefix>v</MinVerTagPrefix>
    <MinVerAutoIncrement>patch</MinVerAutoIncrement>
    
    <!-- NuGet 包配置 -->
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    <PackageId>YourPackage.Name</PackageId>
    <Version>1.0.0</Version>  <!-- 可选，MinVer 会自动覆盖 -->
    <Authors>Your Name</Authors>
    <Description>详细的包描述信息</Description>
    <PackageTags>tag1;tag2;tag3</PackageTags>
    
    <!-- 许可证配置 -->
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    
    <!-- 项目和仓库信息 -->
    <PackageProjectUrl>https://github.com/username/repo</PackageProjectUrl>
    <RepositoryUrl>https://github.com/username/repo.git</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    
    <!-- 包含 README -->
    <PackageReadmeFile>README.md</PackageReadmeFile>
    
    <!-- 可选：包含图标 -->
    <!-- <PackageIcon>icon.png</PackageIcon> -->
    
    <!-- 可选：发布说明 -->
    <!-- <PackageReleaseNotes>Release notes...</PackageReleaseNotes> -->
  </PropertyGroup>

  <ItemGroup>
    <!-- 包含 README 到包中 -->
    <None Include="README.md" Pack="true" PackagePath="\" />
    
    <!-- 包含图标（可选） -->
    <!-- <None Include="icon.png" Pack="true" PackagePath="\" /> -->
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="MinVer" Version="8.0.0-alpha.1">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

</Project>
``` 

### NuGet 元数据属性说明

| 属性 | 必需 | 说明 |
|------|------|------|
| `PackageId` | 是 | 包的唯一标识符 |
| `Version` | 否 | 版本号，MinVer 会自动设置 |
| `Authors` | 是 | 作者信息 |
| `Description` | 是 | 包的描述 |
| `PackageTags` | 否 | 搜索标签，用分号分隔 |
| `PackageLicenseExpression` | 是 | 许可证表达式（如 MIT, Apache-2.0） |
| `PackageProjectUrl` | 否 | 项目主页 URL |
| `RepositoryUrl` | 否 | 源代码仓库 URL |
| `PackageReadmeFile` | 否 | README 文件名 |

### 多目标框架配置

``` xml
<PropertyGroup>
  <!-- 支持多个目标框架 -->
  <TargetFrameworks>net8.0;net9.0;net10.0</TargetFrameworks>
  
  <!-- 或者 Windows 特定框架 -->
  <!-- <TargetFrameworks>net8.0-windows;net9.0-windows</TargetFrameworks> -->
</PropertyGroup>
``` 

### Source Generator 特殊配置

如果你在开发 Roslyn Source Generator，需要额外配置：

``` xml
<PropertyGroup>
  <TargetFramework>netstandard2.0</TargetFramework>
  <LangVersion>latest</LangVersion>
  
  <!-- Source Generator 特定配置 -->
  <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
  <IsRoslynComponent>true</IsRoslynComponent>
  <IncludeBuildOutput>false</IncludeBuildOutput>
  <DevelopmentDependency>true</DevelopmentDependency>
</PropertyGroup>

<ItemGroup>
  <!-- 将编译输出放到 analyzers 目录 -->
  <None Include="$(OutputPath)\$(AssemblyName).dll" 
        Pack="true" 
        PackagePath="analyzers/dotnet/cs" 
        Visible="false" />
</ItemGroup>
``` 

## 三、GitHub Actions 自动化发布

### 完整的 Workflow 配置文件

创建文件 `.github/workflows/publish.yml`：

``` yaml
name: Publish to NuGet

on:
  push:
    tags:
      - 'v*'  # 当推送 v 开头的 tag 时触发
  workflow_dispatch:  # 支持手动触发

env:
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true
  DOTNET_NOLOGO: true

jobs:
  build-and-publish:
    runs-on: windows-latest
    
    steps:
      # 1. 检出代码
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 重要：获取完整的 git 历史，MinVer 需要

      # 2. 获取所有标签（MinVer 需要）
      - name: Fetch all tags
        run: git fetch --tags

      # 3. 显示当前版本（调试用）
      - name: Show current tag
        run: git describe --tags

      # 4. 设置 .NET 环境
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: |
            8.0.x
            9.0.x
            10.0.x

      # 5. 构建项目
      - name: Build
        run: dotnet build --configuration Release -p:ContinuousIntegrationBuild=true

      # 6. 运行测试
      - name: Run tests
        run: dotnet test --configuration Release --no-build --verbosity minimal

      # 7. 打包
      - name: Pack
        run: dotnet pack --configuration Release --no-build --output artifacts

      # 8. 列出生成的包（调试用）
      - name: List packages
        run: Get-ChildItem artifacts -Filter *.nupkg

      # 9. 发布到 NuGet
      - name: Publish to NuGet
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          Get-ChildItem artifacts -Filter *.nupkg | ForEach-Object { 
            dotnet nuget push $_.FullName --source https://api.nuget.org/v3/index.json --api-key ${{ secrets.NUGET_API_KEY }} --skip-duplicate
          }

      # 10. 上传构建产物
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: nuget-packages
          path: artifacts/*.nupkg
          retention-days: 30
``` 

### 多项目打包配置

如果你的解决方案有多个需要发布的项目：

``` yaml
# 分别构建每个项目
- name: Build Core
  run: dotnet build src/MyLib.Core/MyLib.Core.csproj --configuration Release -p:ContinuousIntegrationBuild=true

- name: Build Extension
  run: dotnet build src/MyLib.Extension/MyLib.Extension.csproj --configuration Release -p:ContinuousIntegrationBuild=true

# 分别打包
- name: Pack Core
  run: dotnet pack src/MyLib.Core/MyLib.Core.csproj --configuration Release --no-build --output artifacts

- name: Pack Extension
  run: dotnet pack src/MyLib.Extension/MyLib.Extension.csproj --configuration Release --no-build --output artifacts
``` 

### 配置 NuGet API Key

1. 登录 [nuget.org](https://www.nuget.org/)
2. 进入 Account Settings → API Keys
3. 创建新的 API Key
4. 在 GitHub 仓库中添加 Secret：
   - 进入仓库 Settings → Secrets and variables → Actions
   - 添加名为 `NUGET_API_KEY` 的 secret

### Workflow 关键点解析

#### 1. fetch-depth: 0

``` yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
``` 

**重要**：MinVer 需要完整的 Git 历史来计算版本号，默认的浅克隆（depth=1）会导致版本计算错误。

#### 2. ContinuousIntegrationBuild

``` yaml
run: dotnet build --configuration Release -p:ContinuousIntegrationBuild=true
``` 

这个属性确保源码链接正确生成，对于可调试的 NuGet 包很重要。

#### 3. 条件发布

``` yaml
if: startsWith(github.ref, 'refs/tags/v')
``` 

只在 tag 推送时发布，手动触发时只构建不发布。

## 四、实际项目配置示例

### 项目结构

``` 
MySolution/
├── src/
│   ├── MyLib.Core/
│   │   ├── MyLib.Core.csproj
│   │   └── README.md
│   └── MyLib.Extension/
│       ├── MyLib.Extension.csproj
│       └── README.md
├── .github/
│   └── workflows/
│       └── publish.yml
└── MySolution.sln
``` 

### 完整的 .csproj 示例

**MyLib.Core.csproj**:

``` xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net8.0;net9.0</TargetFrameworks>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    
    <!-- MinVer -->
    <MinVerTagPrefix>v</MinVerTagPrefix>
    <MinVerAutoIncrement>patch</MinVerAutoIncrement>
    
    <!-- NuGet -->
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    <PackageId>MyLib.Core</PackageId>
    <Authors>Your Name</Authors>
    <Description>Core library for MyLib</Description>
    <PackageTags>library;core;dotnet</PackageTags>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <PackageProjectUrl>https://github.com/username/MyLib</PackageProjectUrl>
    <RepositoryUrl>https://github.com/username/MyLib.git</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <PackageReadmeFile>README.md</PackageReadmeFile>
  </PropertyGroup>

  <ItemGroup>
    <None Include="README.md" Pack="true" PackagePath="\" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="MinVer" Version="8.0.0-alpha.1">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

</Project>
``` 

## 五、发布流程

### 日常开发流程

``` bash
# 1. 正常开发提交
git add .
git commit -m "feat: add new feature"

# 2. 推送到远程
git push origin main
``` 

### 发布流程

``` bash
# 1. 确保代码已提交
git status

# 2. 创建并推送 tag
git tag v1.0.0
git push origin v1.0.0

# 3. GitHub Actions 自动执行：
#    - 计算版本号
#    - 构建、测试
#    - 打包
#    - 发布到 NuGet
``` 

### 预发布版本

``` bash
# 创建预发布版本（不会自动发布，除非配置）
git tag v1.1.0-beta.1
git push origin v1.1.0-beta.1
``` 

## 六、最佳实践

### 1. 版本策略

| 版本类型 | 示例 | 使用场景 |
|----------|------|----------|
| 正式版 | `v1.0.0` | 稳定发布 |
| 预发布 | `v1.0.0-beta.1` | 测试版本 |
| 补丁 | `v1.0.1` | Bug 修复 |
| 功能 | `v1.1.0` | 新功能 |
| 重大更新 | `v2.0.0` | 破坏性变更 |

### 2. 提交信息规范

推荐使用约定式提交：

``` 
feat: 添加新功能
fix: 修复 bug
docs: 文档更新
refactor: 代码重构
test: 测试相关
chore: 构建/工具相关
``` 

### 3. 分支策略

``` 
main (或 master)
  ├── develop
  │     ├── feature/xxx
  │     └── fix/xxx
  └── release/x.x.x
``` 

### 4. 安全建议

- 不要在代码中硬编码 API Key
- 使用 GitHub Secrets 存储敏感信息
- 定期轮换 API Key
- 为 API Key 设置适当的权限范围

## 七、常见问题

### Q1: MinVer 版本号不正确？

**原因**：Git 历史不完整

**解决**：
``` yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # 获取完整历史
- run: git fetch --tags  # 获取所有标签
``` 

### Q2: 包发布失败？

**检查清单**：
- [ ] NuGet API Key 是否正确
- [ ] API Key 是否有推送权限
- [ ] 包 ID 是否已被占用
- [ ] 版本号是否已存在

### Q3: 如何在本地测试？

``` bash
# 本地打包
dotnet pack --configuration Release --output ./artifacts

# 本地验证包内容
nuget explorer ./artifacts/YourPackage.1.0.0.nupkg

# 推送到本地 NuGet 服务器测试
dotnet nuget push ./artifacts/YourPackage.1.0.0.nupkg --source http://localhost:5000/v3/index.json
``` 

### Q4: 如何跳过某个项目的发布？

在 `.csproj` 中添加：

``` xml
<PropertyGroup>
  <IsPackable>false</IsPackable>
</PropertyGroup>
``` 

### Q5: Github Action 神秘报错？
我使用cache的时候报错了，如果AI生成的yaml有cache可以试试关闭

## 八、总结

通过 MinVer + NuGet + GitHub Actions 的组合，我们实现了：

1. **版本管理自动化**：基于 Git Tag 自动计算版本号
2. **构建发布自动化**：推送 Tag 即可触发完整流程
3. **可追溯性**：每个版本都有对应的 Git Tag
4. **零配置发布**：无需手动修改版本号或手动打包

这套方案的核心优势：

- **简单**：只需打 Tag 即可发布
- **可靠**：版本号由 Git 历史决定，不会出错
- **自动化**：CI/CD 全自动处理
- **可复用**：配置一次，后续项目直接复用

## 参考资料

- [MinVer 官方文档](https://github.com/adamralph/minver)
- [NuGet 包元数据](https://docs.microsoft.com/nuget/reference/msbuild-targets)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [语义化版本规范](https://semver.org/)
