# 🚀 Fastlane 적용기

## Fastlane이란?

[fastlane - App automation done right](https://fastlane.tools/)

- iOS or 안드로이드앱 빌드 및 배포를 자동화하기 위한 도구
- 스크린샷 생성, 코드사이닝, 빌드 번호 올리기, 릴리즈와 같은 개발 및 릴리즈 워크플로우를 자동화시킴.
- iOS 프로젝트의 경우 애플 개발자 계정이 필요함.

## Fastlane 설치

### 공식 문서

[https://docs.fastlane.tools/getting-started/ios/setup/](https://docs.fastlane.tools/getting-started/ios/setup/)

<aside>
💡 Xcode command line tools, Ruby 등을 먼저 설치해야 하지만, Xcode가 이미 설치된 환경에서는 이미 설치돼 있으므로 패스!
* 빌드 서버로 구축된 경우에는 설치하면 될 듯

</aside>

### Bundler 설치

```bash
gem install bundler
```

- fastlane 업데이트시 쓰인다고 함.

### Fastlane 설치

```bash
brew install fastlane
```

- homebrew를 이용해서 설치

### Fastlane 초기화

```bash
% cd Desktop/MyProject
% fastlane init

# 애플 개발자 계정 정보 입력
[13:07:37]: Please enter your Apple ID developer credentials
[13:07:37]: Apple ID Username:
myaccount@gmail.com

# 패스워드도 입력
Password (for myaccount@gmail.com): ***********

# 인증코드 6자리 입력
Please enter the 6 digit code:
000000
```

- 프로젝트 파일이 있는 경로까지 이동후에 fastlane을 초기화 해준다.
- 아래와 같은 파일이 생성됨
    
    ![스크린샷 2024-03-14 오후 4.10.16.png](Images/fastlane_1.png)
    

### 배포 명령 실행

```bash
% fastlane lane_name

# Fastfile 내부에 아래와 같이 lane 이름이 지정되어 있음. 변경 가능.
# lane :lane_name do
```

- Fastfile내부를 보면 아래와 같은데 용도에 따라 lane을 별도로 구성할 수 있다.
- 명령어 입력시 Step 별로 진행됨.

### 배포 명령 실행중 발생한 에러(시간 순서로 정리됨)

- 💥 build_app
    - 프로파일이 없다는 에러
        
        ```bash
        2024-03-14 13:30:02.006 xcodebuild[7582:131838]  IDEDistribution: App Store Connect request for store configuration failed for account (null) (Account "(null)": Unable to authenticate with App Store Connect (Error Domain=ITunesConnectionOperationErrorDomain Code=1085 "No provider associated with App Store Connect user" UserInfo={NSLocalizedRecoverySuggestion=No provider associated with App Store Connect user, NSLocalizedFailureReason=App Store operation failed., NSLocalizedDescription=No provider associated with App Store Connect user}))
        error: exportArchive: No profiles for 'com.appidtest.myapp' were found
        
        Error Domain=IDEProfileLocatorErrorDomain Code=1 "No profiles for 'com.appidtest.myapp' were found" UserInfo={IDEDistributionIssueSeverity=3, NSLocalizedDescription=No profiles for 'com.appidtest.myapp' were found, NSLocalizedRecoverySuggestion=Xcode couldn't find any iOS App Store provisioning profiles matching 'com.appidtest.myapp'. Automatic signing is disabled and unable to generate a profile. To enable automatic signing, pass -allowProvisioningUpdates to xcodebuild.}
        
        - * EXPORT FAILED **
        [13:30:09]: Exit status: 70
        ```
        
        - 자동으로 사이닝 되도록 설정 필요했지만 해당 옵션을 주지 않았음
        
        ```bash
        lane :beta do
          build_app(xcargs: "-allowProvisioningUpdates")
        end
        ```
        
        - 위와 같이 `allowProvisioningUpdates` 옵션 추가
- 💥 upload_to_testflight
    - authorization에 실패했다는 에러
        
        ```bash
        [14:12:03]: fastlane finished with errors
        
        [!] Error uploading ipa file: 
         [Application Loader Error Output]: Error uploading '/var/folders/kd/k1f6s0qn5fqg2_lqdrh40xfw0000gn/T/e4ec9125-3bc7-4eae-97a5-28435f595968.ipa'.
        [Application Loader Error Output]: Unable to upload archive. Failed to get authorization for username 'myaccount@gmail.com' and password. (
        [Application Loader Error Output]: The call to the altool completed with a non-zero exit status: 1. This indicates a failure.
        ```
        
        - App Store Connect API 액세스를 요청해야 함.
            - App Store Connect → 사용자 및 액세스 → 통합 → API Key 생성
            
            ![스크린샷 2024-03-14 오후 4.41.06.png](Images/fastlane_2.png)
            
            - 키를 생성하고 나면 인증서(.p8) 다운로드가 가능하고 해당 인증서내에 Private key가 있음
            - json file을 만들어서 아래와 같이 설정([https://docs.fastlane.tools/app-store-connect-api/#using-fastlane-api-key-json-file](https://docs.fastlane.tools/app-store-connect-api/#using-fastlane-api-key-json-file))
            
            ```bash
            # 인증서 파일 내에 key 내용 그대로 복사(개행되어 있는 부분은 하나도 빼놓지 않고 개행문자 '\n' 붙여줘야 함!)
            # issuer_id는 App Store Connect내에서 확인 가능함.
            # fastlane/D383SF739.json
            {
              "key_id": "D383SF739",
              "issuer_id": "6053b7fe-68a8-4acb-89be-165aa6465141",
              "key": "-----BEGIN PRIVATE KEY-----\nMIGTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHknlhdlYdLu\n-----END PRIVATE KEY-----",
              "duration": 1200, # optional (maximum 1200)
              "in_house": false # optional but may be required if using match/sigh
            }
            
            =======================================================================================
            
            # 위와 같이 json 파일 추가했으면 Fastfile에 참조 걸어줌
            # fastlane/Fastfile
            lane :release do
              pilot( api_key_path: "fastlane/D383SF739.json" )
            end
            ```
            
- 💥 pilot
    - `pilot`명령어는 `upload_to_testflight`의 별칭이므로 `upload_to_testflight`를 사용할 필요가 없음.
    - invalid curve name
        - App Store Connect API Key를 등록했는데도 에러가 발생함
        
        ```bash
        /usr/local/Cellar/fastlane/2.219.0_2/libexec/gems/fastlane-2.219.0/spaceship/lib/spaceship/connect_api/token.rb:71:in `initialize': [!] invalid curve name (OpenSSL::PKey::ECError)
        ```
        
        - 다운로드 받은 App Store Connect API 인증서(.p8)의 내용중 개행이 된 부분들에 ‘\n’ 안붙이고 json 에 통째로 붙여 넣어서 문제가 됨.
            - json 파일의 key 항목에서 개행된 부분에는 모조리 ‘\n’ 개행 문자 넣어줘야 함.
            - 개행문자까지도 Key의 일부라고 보면 될 듯.
            
            ```bash
            {
              "key_id": "D383SF739",
              "issuer_id": "6053b7fe-68a8-4acb-89be-165aa6465141",
              "key": "-----BEGIN PRIVATE KEY-----\nMIGTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHknlhdlYdLu\n-----END PRIVATE KEY-----",
              "duration": 1200, # optional (maximum 1200)
              "in_house": false # optional but may be required if using match/sigh
            }
            ```

    - No ipa or pkg file given
        - 빌드를 안하고 pilot 실행해서 벌어지는 문제..
        - 빌드부터 해야됨. 순서가 있음.
    
        ```bash
        default_platform(:ios)
        
        platform :ios do
          desc "Push a new beta build to TestFlight"
          lane :myapp_cicd_test do
            build_app(scheme: "MyApp_Debug", xcargs: "-allowProvisioningUpdates")
        
            pilot( api_key_path: "fastlane/3GHVD43H9X.json" )
            
            upload_to_testflight
          end
        end
        ```
            

### Testfligt 배포 완료

![스크린샷 2024-03-14 오후 5.02.32.png](Images/fastlane_3.png)

![스크린샷 2024-03-14 오후 5.04.58.png](Images/fastlane_4.png)

### 참고자료

- [https://gyuios.tistory.com/241](https://gyuios.tistory.com/241)
- [https://vapor3965.tistory.com/95](https://vapor3965.tistory.com/95)
