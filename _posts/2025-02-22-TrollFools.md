---
layout: post
title: "TrollFools"
excerpt_separator: <!--more-->
date: 2025-02-22 00:00:00 +0800
author: xiangxiang
categories: security
tags: [iOS security]
---
iOS injection without JB

<!--more-->
* auto-gen TOC:
{:toc}

## 0x00 Exploit Needed for JB
### kernel r/w
userspace -> kernel space
- tfp0: iOS 14 tfp0 hardening
- kmsg < iOS14.2 : convert a two-byte heap overflow (inkalloc_large area) to a full exploit
- CVE-2021-1782 < iOS 14.4: A race condition in `user_data_get_value()` leading to ivac entry UAF
  + [CVE-2021-1782](https://www.synacktiv.com/publications/analysis-and-exploitation-of-the-ios-kernel-vulnerability-cve-2021-1782)
  + [https://github.com/ModernPwner/cicuta_virosa](https://github.com/ModernPwner/cicuta_virosa)
- IOSurface
  + [IOSurface](https://powerofcommunity.net/poc2019/Liang.pdf)
- kfd
  + [https://github.com/felix-pb/kfd](https://github.com/felix-pb/kfd)
  
### Integrity bypass
- [https://support.apple.com/guide/security/operating-system-integrity-sec8b776536b/web](https://support.apple.com/guide/security/operating-system-integrity-sec8b776536b/web)
- PPL stands for Page Protection Layer, and it works as a means of security by preventing code from being modified once it’s verified by the system
- PAC stands for Pointer Authentication Codes, 
  + [https://ivrodriguez.com/pointer-authentication-on-armv8-3/amp/](https://ivrodriguez.com/pointer-authentication-on-armv8-3/amp/)
- We need kernel PAC bypass to do the kcall things, or PPL bypass
  + [https://bazad.github.io/presentations/BlackHat-USA-2020-iOS_Kernel_PAC_One_Year_Later.pdf](https://bazad.github.io/presentations/BlackHat-USA-2020-iOS_Kernel_PAC_One_Year_Later.pdf)


```text

+-------+   ---->  Trust Cache  ---->  Core Trust  ----> amfid
| fork/ |             |                                    |
| exec  |             |                                    |
+-------+             +---------------         --------->  +------> a.out
              

```

- TrustCache
- CoreTrust

## 0x01 TrollStore
### 1.1 Code signature
- [https://theapplewiki.com/wiki/Code_signature](https://theapplewiki.com/wiki/Code_signature)

Inside Code Signing series
- [https://developer.apple.com/documentation/technotes/tn3125-inside-code-signing-provisioning-profiles](https://developer.apple.com/documentation/technotes/tn3125-inside-code-signing-provisioning-profiles)
- [https://developer.apple.com/documentation/technotes/tn3126-inside-code-signing-hashes](https://developer.apple.com/documentation/technotes/tn3126-inside-code-signing-hashes)
- [https://developer.apple.com/documentation/technotes/tn3127-inside-code-signing-requirements](https://developer.apple.com/documentation/technotes/tn3127-inside-code-signing-requirements)
- [https://developer.apple.com/documentation/technotes/tn3161-inside-code-signing-certificates](https://developer.apple.com/documentation/technotes/tn3161-inside-code-signing-certificates)

### 1.2 Exploits in TrollStore1
- [Slide](/img/OBTS_v5_lHenze.pdf)
- [Fugu15 Presentation Video](https://www.youtube.com/watch?v=rPTifU1lG7Q)
- [Write-Up on the first CoreTrust bug with more information](https://worthdoingbadly.com/coretrust/)

### 1.3 CVE-2023-41991
- [ChOma](https://github.com/opa334/ChOma) is a simple library for parsing and manipulating MachO files and their CMS blobs. Written for exploitation of CVE-2023-41991

- CoreTrust was found to improperly validate CMS blobs that had multiple signers. By including a CMS signature from an App Store binary, CoreTrust returns the CodeDirectory hashes to AMFI from the fake signer, but use the App Store signer to decide whether or not the binary is an App Store binary

- As a result, binaries can be permanently signed on device and have **arbitrary entitlements**, apart from a [few restricted ones](https://github.com/opa334/TrollStore?tab=readme-ov-file#banned-entitlements) that are only allowed to be used by trustcached binaries.

- The vulnerability is caused by CoreTrust incorrectly handling multiple signers in a CMS signature blob. The signature blob will have two signers: 
1. the App Store certificate chain (which has a valid signature for a different code signature) 
2. a custom certificate chain (which has a valid signature for our code signature). 

- Due to it incorrectly validating both signers, CoreTrust will return the CD hashes from our signer but set the policy flags using the App Store signer.

- The exploit is implemented in `ct_bypass`, and works by:
1. Updating the load commands by calculating the new sizes of the `__LINKEDIT` segment and the code signature.
2. Updating the page hashes in the SHA256 CodeDirectory to match the new load command data.
3. Replacing the SHA1 CodeDirectory with one from a valid App Store binary.
4. Creating a new signature blob that has two signers, the App Store and our custom certificate chain.
5. Updating the necessary fields in the signature blob to match the CD hashes.
6. Signing the signature blob for the custom identity (the App Store identity will already have an intact signature).
7. Inserting the new code signature into the binary.

### 1.4 TrollStore2
- It works because of an AMFI/CoreTrust bug where iOS does not correctly verify code signatures of binaries in which there are multiple signers (CVE-2023-41991)
- The `com.apple.private.persona-mgmt` entitlement allows a binary to spawn another binary under the root user (or any user, for that matter)


## 0x02 TrollFools
- In-place tweak injection with insert_dylib and ChOma.
Overview
- chown & ct_bypass `tweak.dylib`
- cp `tweak.dylib` to `@executable_path/Frameworks/`
- Choose a framework machO (or main machO 高级选项中的优先处理主要可执行文件) to inject
- backup targetMachO
- `insert_dylib tweak.dylib targetMachO`
- `ldid -S targetMachO`
- `install_name_tool -change tweak.dylib @rpath/tweak.dylib targetMachO`
- `ct_bypass -r -i targetMachO -t TeamId`
- chown


`otool -l machO` to verify

### 2.1 list apps that can be injected
- Frameworks Directory exists
- `TrollFools/AppListModel.swift` -> `InjectorV3.main.checkIsEligibleAppBundle`

```swift
// Frameworks Directory exists
func checkIsEligibleAppBundle(_ target: URL) -> Bool {
    guard checkIsBundle(target) else {
        return false
    }

    let frameworksURL = target.appendingPathComponent("Frameworks")
    return !((try? FileManager.default.contentsOfDirectory(at: frameworksURL, includingPropertiesForKeys: nil).isEmpty) ?? true)
}
```

### 2.2 TrollFools/InjectorV3+Inject.swift

```swift
func inject(_ assetURLs: [URL]) throws {
    let preparedAssetURLs = try preprocessAssets(assetURLs)

    precondition(!preparedAssetURLs.isEmpty, "No asset to inject.")
    terminateApp()

    try injectBundles(preparedAssetURLs
        .filter { $0.pathExtension.lowercased() == "bundle" })

    try injectDylibsAndFrameworks(preparedAssetURLs
        .filter { $0.pathExtension.lowercased() == "dylib" || $0.pathExtension.lowercased() == "framework" })
}
```

```bash
ct_bypass -r -i /var/mobile/Library/Caches/wiki.qaq.TrollFools/InjectorV3/56A6617A-FB1C-4DED-A452-01609990B354/demo.dylib -t ZMX4C3B995

chown 33:33 /var/mobile/Library/Caches/wiki.qaq.TrollFools/InjectorV3/56A6617A-FB1C-4DED-A452-01609990B354/demo.dylib

cp-15 -rfp /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy.troll-fools.bak

rm -rf /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/demo.dylib

cp-15 --reflink=auto -rfp /var/mobile/Library/Caches/wiki.qaq.TrollFools/InjectorV3/56A6617A-FB1C-4DED-A452-01609990B354/demo.dylib /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/demo.dylib

chown 33:33 /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/demo.dylib

# @executable_path/Frameworks already in rpath
## install_name_tool -add_rpath @executable_path/Frameworks /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy 


insert_dylib @rpath/demo.dylib /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy --inplace --overwrite --no-strip-codesig --all-yes --weak
# Added LC_LOAD_WEAK_DYLIB to /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy

ldid -S /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy

install_name_tool -change @rpath/demo.dylib @rpath/demo.dylib /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy

ct_bypass -r -i /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy -t ZMX4C3B995

chown 33:33 /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy
```

### 2.3 TrollFools/InjectorV3+Eject.swift
```swift
func eject(_ assetURLs: [URL]) throws {
    precondition(!assetURLs.isEmpty, "No asset to eject.")
    terminateApp()

    try ejectBundles(assetURLs
        .filter { $0.pathExtension.lowercased() == "bundle" })

    try ejectDylibsAndFrameworks(assetURLs
        .filter { $0.pathExtension.lowercased() == "dylib" || $0.pathExtension.lowercased() == "framework" })
}
```


```bash
optool uninstall -p @rpath/demo.dylib -t /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy
# Successfully removed all entries for @rpath/demo.dylib

# Writing executable to /var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy...
rm -f /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/demo.dylib
ct_bypass -r -i /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy -t ZMX4C3B995

chown 33:33 /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy

rm -rf /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy

mv-15 -f /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy.troll-fools.bak /private/var/containers/Bundle/Application/B21CD6F2-5D73-4EC1-B5FE-745688DEE9CB/EMBS.app/Frameworks/AIPIFlyMSCDy.framework/AIPIFlyMSCDy

```

## 0x03 TODO: opainject

## 0x04 More
- [https://juejin.cn/post/6844904190968332302](https://juejin.cn/post/6844904190968332302)
- [kfd exploit](https://github.com/felix-pb/kfd)