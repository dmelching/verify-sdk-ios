# IBM Security Verify MFA SDK for iOS

The MFA software development kit (SDK) enables applications to register authenticators and enroll into the associated factors.


## Example
An [example](../../Examples/mfa) application is available for the MFA SDK

## Getting started

### Installation

[Swift Package Manager](https://swift.org/package-manager/) is used for automating the distribution of Swift code and is integrated into the `swift` compiler.  To depend on one or more of the components, you need to declare a dependency in your `Package.swift`:

```swift
dependencies: [
    .package(name: "IBM Security Verify", url: "https://github.com/ibm-security-verify/verify-sdk-ios.git", from: "3.0.4")
]
```

then in the `targets` section of the application/library, add one or more components to your `dependencies`, for example:

```swift
// Target for Swift 5.7
.target(name: "MyExampleApp", dependencies: [
    .product(name: "MFA", package: "IBM Security Verify")
],
```

Alternatively, you can add the package manually.
1. Select your application project in the **Project Navigator** to display the configuration window.
2. Select your application project under the **PROJECT** heading
3. Select the **Swift Packages** tab.
4. Click on the `+` button.
5. Enter `https://github.com/ibm-security-verify/verify-sdk-ios.git` as the respository URL and follow the remaining steps selecting the components to add to your project.

### API documentation
The MFA SDK API can be reviewed [here](https://ibm-security-verify.github.io/ios/documentation/mfa/).

### Importing the SDK

Add the following import statement to the `.swift` files you want to reference the MFA SDK.

```swift
import MFA
```

## Usage

### Register a multi-factor authenticator

To register an authenticator, first obtain the JSON string value from the QR code generated by IBM Security Verify or IBM Security Verify Access.

```swift
let controller = MFARegistrationController(json: qrScanResult)

// Initiate the registration provider.
let provider = try await controller.initiate(with: "John Doe", pushToken: "abc123")

// Get the next enrollable signature.
guard let factor = await provider.nextEnrollment() else {
   return
}

// Create the key-pair using default SHA512 hash.
let key = RSA.Signing.PrivateKey()
let publicKey = key.publicKey

// Sign the data with the private key.
let value = factor.dataToSign.data(using: .utf8)!
let signature = try key.signature(for: value)

// Add to the Keychain.
try KeychainService.default.addItem("biometric", value: key.derRepresentation, accessControl: factor.biometricAuthentication ? .biometryCurrentSet : nil)
    
// Enroll the factor.
try await provider.enroll(with: "biometric", publicKey: key.publicKey.x509Representation signedData: String(decoding: signature.rawRepresentable, as: UTF8.self)

// NOTE: Alternatively instead of creating the keys and saving to the Keychain, you could simply call:
try await provider.enroll()

// Complete the registration and save authenticator.
let authenticator = try await provider.finalize()
let data = try JSONEncoder().encode(authenticator)
try data.write(to: "fileLocation")
```

The private key saved to the Keychain can be accessed using `authenticator.id.<type>` where the type is either `userPresence` or `fingerprint`.  For example:
```
guard let privateKeyData = try? KeychainService.default.readItem("\(authenticator.id).userPresence"),
    let privateKey = try RSA.Signing.PrivateKey(derRepresentation: privateKeyData) else {
    // No item found
    return
}
```

### Check for a transaction

Checking for a transaction requires an instance of the `MFAAuthenticatorService`.  The service maintains the current pending transaction therefore the instance must be retained between calls to `nextTransaction` and `completeTransaction`.

This example demonostrates how to check for a transaction:

```swift
// Create the controller and initiate the service.
let controller = MFAServiceController(using: authenticator)
let service = controller.initiate()

let transaction = try await service.nextTransaction(with: nil)
print(transaction)
```

### Approve or deny a transaction

Continuing the previous code snippet, the following example demonstrates how to deny or approve a transaction:

```swift
// Deny
if let factorType = authenticator.allowedFactors.first(where: {$0.id == transaction.factorID } {
   do {
      try await self.service.completeTransaction(action: .deny, factor: factorType)
   }
   catch let error {
      print(error)
   }  
}

// Approve
if let factorType = authenticator.allowedFactors.first(where: {$0.id == transaction.factorID } {
   do {
      try await self.service.completeTransaction(factor: factorType)
   }
   catch let error {
      print(error)
   }  
}
```

## License
This package contains code licensed under the MIT License (the "License"). You may view the License in the [LICENSE](../../LICENSE) file within this package.