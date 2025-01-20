# Troubleshooting for the Tuist

## 초기화(`tuist init`)

1. `tuist init --platform ios --template swiftui`
    1. 에러 발생!
    Can't initialize a project in the non-empty directory at path /Users/taeyoungson/Desktop/SpeechCard.
    Consider creating an issue using the following link: https://github.com/tuist/tuist/issues/new/choose
    2. `tuist init`을 시도하는 디렉터리는 비어있어야 한다! readme 파일이 존재해서 에러가 발생했으므로 파일 제거 후 재시도 → 성공!

## 프로젝트 구성(`tuist generate`)

### 🚨 Errors

- **`Unable to load manifest at /Users/taeyoungson/Desktop/SpeechCard/Project.swift`**
    
    ```xml
    ├── Plugins
    │   └── SpeechCard
    │       ├── Package.swift
    │       ├── Plugin.swift
    │       ├── ProjectDescriptionHelpers
    │       │   └── LocalHelper.swift
    │       └── Sources
    │           └── tuist-my-cli
    │               └── main.swift
    ├── README.md
    ├── SpeechCardApp
    │   ├── Project.swift
    │   └── Sources
    ├── Targets
    │   ├── SpeechCard
    │   │   ├── Resources
    │   │   │   ├── Assets.xcassets
    │   │   │   │   ├── AccentColor.colorset
    │   │   │   │   │   └── Contents.json
    │   │   │   │   ├── AppIcon.appiconset
    │   │   │   │   │   └── Contents.json
    │   │   │   │   └── Contents.json
    │   │   │   └── Preview Content
    │   │   │       └── Preview Assets.xcassets
    │   │   │           └── Contents.json
    │   │   ├── Sources
    │   │   │   └── SpeechCardApp.swift
    │   │   └── Tests
    │   │       └── SpeechCardTests.swift
    │   ├── SpeechCardKit
    │   │   ├── Sources
    │   │   │   └── SpeechCardKit.swift
    │   │   └── Tests
    │   │       └── SpeechCardKitTests.swift
    │   └── SpeechCardUI
    │       ├── Sources
    │       │   └── ContentView.swift
    │       └── Tests
    │           └── SpeechCardUITests.swift
    ├── Tuist
    │   ├── Config.swift
    │   └── ProjectDescriptionHelpers
    │       └── Project+Templates.swift
    └── ~~Project.swift~~ => Workspace.swift
    ```
    
    - SpeechCardApp(앱 프로젝트)를 별도의 모듈로 뺐으므로 Root 에서 Project.swift가 아닌 Workspace.swift로 변경하고 SpeechCardApp을 프로젝트로 추가해야 함!
    
    ```swift
    //  Workspace.swift
    
    import ProjectDescription
    
    let workspace = Workspace(
        name: "SpeechCard",
        projects: [
            "SpeechCardApp/**"
        ]
    )
    ```
    
- **`The target SpeechCardApp has the following invalid resource globs: "/Users/taeyoungson/Desktop/SpeechCard/SpeechCardApp/Resources/**" does not exist.`**
    - Resources 디렉토리가 없다는 뜻!
    - 각 프로젝트는 프로젝트의 타겟에서 설정했던 sources 및 resources 경로를 준수하도록 파일 트리를 구성해야 함.
    
    ```swift
    // ...
    
    switch $0 {
    case .app:
        target = Target(
            name: name,
            platform: platform,
            product: .app,
            bundleId: "io.tuist.\(name)",
            infoPlist: .extendingDefault(with: infoPlist),
            sources: ["Sources/**"],
            resources: ["Resources/**"],
            dependencies: dependencies
        )
    
    // ...
    ```
    
    - Sources, Resources, Tests.. 등등의 디렉토리엔 최소 1개 이상의 swift파일이 포함되어야 Xcode 내비게이터에서 보여진다.
    (빈 디렉토리면 프로젝트 생성엔 성공하나 내비게이터에 디렉토리가 보이지 않는다.)
    
    ```xml
    ...
    ├── SpeechCardApp
    │   ├── Derived
    │   │   ├── InfoPlists
    │   │   │   ├── SpeechCardApp-Info.plist
    │   │   │   ├── SpeechCardAppTests-Info.plist
    │   │   │   └── SpeechCardAppUITests-Info.plist
    │   │   └── Sources
    │   │       └── TuistBundle+SpeechCardApp.swift
    │   ├── Project.swift
    │   ├── Resources
    │   │   └── empty.swift
    │   ├── Sources
    │   │   └── empty.swift
    │   ├── SpeechCardApp.xcodeproj
    │   │   ├── project.pbxproj
    │   │   ├── project.xcworkspace
    │   │   │   └── contents.xcworkspacedata
    │   │   ├── xcshareddata
    │   │   │   └── xcschemes
    │   │   │       └── SpeechCardApp.xcscheme
    │   │   └── xcuserdata
    │   │       └── taeyoungson.xcuserdatad
    │   │           └── xcschemes
    │   │               └── xcschememanagement.plist
    │   ├── Tests
    │   │   └── empty.swift
    │   └── UITests
    │       └── empty.swift
    ...
    ```
    

### ⚠️ Warnings

- **`Target 'CombineSchedulers' has been linked from target 'Challenge', target 'Shelf', it is a static product so may introduce unwanted side effects.`**
    - CombineSchedulers(TCA의 하위 모듈. 즉, Third Party 라이브러리의 모듈임.) 모듈이 각각의 Target에 모두 정적으로 링크되었음을 경고함.
        - 정적으로 여러 모듈에 링크되면 모든 모듈 각각이 Third Party 라이브러리의 코드를 복사하므로 비효율이 발생함.
    - 동적 링크(framework)로 해결
        
        ```swift
        // Package.swift
        
        import PackageDescription
        
        // 빌드 세팅 커스텀(https://docs.tuist.dev/ko/guides/develop/projects/dependencies#external-dependencies)
        #if TUIST
            import ProjectDescription
            import ProjectDescriptionHelpers
        
            let packageSettings = PackageSettings(
                productTypes: [
                    "ComposableArchitecture": .framework, // default is .staticFramework
                ]
            )
        
        #endif
        
        let package = Package(
            name: "SpeechCard",
            dependencies: [
                .package(
                    url: "https://github.com/pointfreeco/swift-composable-architecture",
                    .upToNextMajor(from: "1.6.0")
                )
            ],
            targets: [
                .executableTarget(name: "SpeechCard")
            ]
        )
        ```
        
        - `PackageSettings`
            - 패키지 통합 방식을 설정할 수 있음.
            - ComposableArchitecture를 framework(동적)로 링킹할 수 있음.
