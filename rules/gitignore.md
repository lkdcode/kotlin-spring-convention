# rules/gitignore — .gitignore 템플릿

프로젝트 루트에 위치. 아래 내용을 `.gitignore` 로 사용.

```gitignore
# ===========================
# IDE — IntelliJ IDEA
# ===========================
.idea/*
!.idea/codeStyles
!.idea/runConfigurations

.idea/**/workspace.xml
.idea/**/tasks.xml
.idea/**/usage.statistics.xml
.idea/**/dictionaries
.idea/**/shelf
.idea/**/aws.xml
.idea/**/contentModel.xml
.idea/**/dataSources/
.idea/**/dataSources.ids
.idea/**/dataSources.local.xml
.idea/**/sqlDataSources.xml
.idea/**/dynamic.xml
.idea/**/uiDesigner.xml
.idea/**/dbnavigator.xml
.idea/**/gradle.xml
.idea/**/libraries
.idea/**/mongoSettings.xml
.idea/replstate.xml
.idea/sonarlint/
.idea/httpRequests
.idea/caches/build_file_checksums.ser
.idea_modules/
atlassian-ide-plugin.xml
*.iws
cmake-build-*/

# ===========================
# IDE — VS Code
# ===========================
.vscode/*
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
!.vscode/*.code-snippets
.history/
.history
.ionide
*.vsix

# ===========================
# Gradle
# ===========================
.gradle
**/build/
!src/**/build/
gradle-app.setting
!gradle-wrapper.jar
!gradle-wrapper.properties
.gradletasknamecache
*.hprof

# ===========================
# Java / Kotlin
# ===========================
*.class
*.log
*.ctxt
.mtj.tmp/
*.jar
*.war
*.nar
*.ear
*.zip
*.tar.gz
*.rar
hs_err_pid*
replay_pid*

# ===========================
# 코드 생성 (QueryDSL / jOOQ)
# ===========================
src/main/generated/
generated/

# ===========================
# 민감 정보 — 절대 커밋 금지
# ===========================
gradle.properties
src/main/resources/local/
src/main/resources/dev/secret/
src/main/resources/prod/secret/
*.p12
*.jks
*.pem
*.key
*-firebase-adminsdk-*.json

# ===========================
# macOS
# ===========================
.DS_Store
.AppleDouble
.LSOverride
Icon
._*
.DocumentRevisions-V100
.fseventsd
.Spotlight-V100
.TemporaryItems
.Trashes
.VolumeIcon.icns
.com.apple.timemachine.donotpresent
.AppleDB
.AppleDesktop
Network Trash Folder
Temporary Items
.apdisk
*.icloud

# ===========================
# Windows
# ===========================
Thumbs.db
Thumbs.db:encryptable
ehthumbs.db
ehthumbs_vista.db
*.stackdump
[Dd]esktop.ini
$RECYCLE.BIN/
*.cab
*.msi
*.msix
*.msm
*.msp
*.lnk

# ===========================
# Maven
# ===========================
target/
pom.xml.tag
pom.xml.releaseBackup
pom.xml.versionsBackup
pom.xml.next
release.properties
dependency-reduced-pom.xml
buildNumber.properties
.mvn/timing.properties
.mvn/wrapper/maven-wrapper.jar
.project
.classpath

# ===========================
# Git Worktrees
# ===========================
.worktrees/
```

## 반드시 커밋 금지 항목

| 항목 | 이유 |
|---|---|
| `gradle.properties` | Flyway DB 접속 정보 |
| `src/main/resources/*/secret/` | JWT secret-key, AWS key, NCP key 등 |
| `*-firebase-adminsdk-*.json` | FCM 서비스 계정 키 |
| `*.pem`, `*.key`, `*.p12`, `*.jks` | 인증서, 개인키 |

## IntelliJ 공유 허용 항목

```gitignore
!.idea/codeStyles        ← 코드 스타일 팀 공유
!.idea/runConfigurations ← 실행 설정 팀 공유
```
