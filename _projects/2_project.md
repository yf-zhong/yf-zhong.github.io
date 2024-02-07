---
layout: page
title: Cryptographic
description: End-to-End Encrypted File-Sharing System
img: assets/img/e2e_fig.webp
importance: 2
category: Course Project
giscus_comments: true
---

## Overview

This project introduces a client application for a secure file sharing system, leveraging various cryptographic primitives to ensure data security. Imagine something similar to Dropbox, but secured with cryptography so that the server cannot view or tamper with user data.

The client is implemented in **Golang** and offers a suite of features including:

- Authenticate with a username and password;
- Save files to the server;
- Load saved files from the server;
- Overwrite saved files on the server;
- Append to saved files on the server;
- Share saved files with other users; and
- Revoke access to previously shared files.

This project showcases how cryptography can be wielded to bolster traditional file-sharing systems, providing both utility and peace of mind to its users. Dive into the code to explore the cryptographic techniques employed and how they fortify the entire system against potential security threats.

## Data Structure

```go
type User struct {
    Username           string // store for future use (maybe)
    UserBaseKey        []byte
    UserRSAPrivateKey  PKEDecKey // Public key --> username + "RSA_Public_Key"
    UserSignPrivateKey DSSignKey // Public key --> username + "Digital_Signature_Key"
}

type FileHeader struct { // UUID <-- "username_filename"
    ShareId   UUID   // the ShareNode for a user
    FHBaseKey []byte // protect direct ShareNode
}

type ShareNode struct { // root UUID <-- random, child UUID <-- "sendername_recipientname_filename"
    FileBodyId   UUID
    ShareNodeId  UUID     // Used in CreateInvitation
    LockboxId    UUID     // Invitation is also the lockbox
    SNBaseKey    []byte   // Protect following ShareNode if is root, protect itself otherwise
    ChildrenName []string // Children's name list for the root ShareNode, nil if is child
}

type FileBody struct { // UUID <-- random
    FBBaseKey   []byte // Protect all file contents
    LastContent UUID   // Store the last content being appended into the file
}

type FileContent struct { // UUID <-- random
    Content     []byte
    PrevContent UUID
}

type Invitation struct {
    ShareId           UUID   // Recipient ShareNode location
    InvitationBaseKey []byte // Protect lockbox
}

type Lockbox struct {
    FileBaseKey []byte // Protect file
}

type SymEncData struct {
 CypherText []byte
 MAC        []byte
}

type PublicEncData struct {
 CypherText []byte // RSA Encrypted Invitation data
 Signature  []byte
}
```

## Implementation Details

### User Authentication

> User Authenticatoin will be performed with multi-level protection using symmetric encryption and MAC.
>
> Since `User` will not store any File specified data, and `FileHeader` will be retrieved each time we want to access a file, multiple user instance is automatically supported.

#### `InitUser(username string, password string)`

- User authentication will be done on fly when the user enter username and password. A user base key will be generated from the `username||password` using PBKDF.

  - Generate UUID from `username` using `getUUIDFromString()` and check that the UUID does't exist in _DataStore_
  - Get the root base key from `usename||password` using `getUserBaseKey()`. Then, derive the root encryption/MAC key from root base key using `getKeyPairFromBase()`. This key pair is used to protect the `User` itself.
  - Generate user base key from root base key using `getNextBaseKey()` and store in `User`.
  - Generate RSA key pair and digital signature key pair. The public keys will be stored in _KeyStore_ and the private keys will be stored in `User`.
  - Save the `User` in to _DataStore_ using root Enc/MAC keys by calling `storeObject()`.

#### `GetUser(username string, password string)`

- Generate UUID from username using `getUUIDFromString()`, and generate root base key from`generateUserBaseKey()`.Then derive root Enc/Mac Keys.
- Get `User` from _DataStore_ with Enc/MAC Keys using `getObject()`.

### File Storage and Retrieval

> File is stored using a multilevel structure: {FileHeader} -- {ShareNode} -- {FileBody} -- {FileContent}
>
> The top level `FileHeaderNode` is protected by `UserBaseKey`. `FileHeader` UUID is derived from `username||filename` in the userspace on fly. It contains `ShareNode` UUID of direct owner `ShareNode`. The base key here is derived from `User` base key.
>
> The collection of `ShareNode` is a flattened tree of height 2. Each `ShareNode` stores the following FileBody and self's UUID, the `Lockbox` containing the base key to protect `FileBody`, a ShareNode base key to derive child and lock base keys, and a children name array. The array will be nil if the `ShareNode` is a child, and empty if the `ShareNode` is a root.
>
> `FileBody` contains the UUID for the last appended `FileContent` and a base key to protect all the `FileContent`. The UUID of FileBody is generated randomly.
>
> `FileContent` contains the actual contents and the UUID for the previous content being appended. The UUID of `FileContent` is generated randomly.
>
> Using `LastContent` in `FileBody` and `PrevContent` in `FileContent`, we established a linked file content structure.
>
> Since `FileContent` is appended indivitually into the content list and we only need to update two UUID, efficiency is guaranteed for `AppendToFile()`

#### `StoreFile(filename string, content []byte)`

- Based on the file structure described above, we create and store `FileHeader`, `ShareNode`, `FileBody`, and `FileContent`, and generate base key for each layer. Base key is used to drive keys to protect sub-layers. Note that key protecting `FileBody` will be put into `Lockbox`.
- When `StoreFile` is called, the receiver will always be the owner of the file. Thus, we will set `ChildrenName` to empty.
- `LastContent` in `FileBody` will be set to the `FileContent` created in `StoreFile()`.

#### `LoadFile(filename string)`

- First get `FileBody` by calling `getFileBody()`, get the `LastContent` in `FileBody`, and retrieve that `FileContent`.
- Repeatedly retrieve `FileContent` using `PrevContent` in `FileContent` until `PrevContent` is `uuid.Nil`. Every time we retrieve a new FileContent, we will appended the content in `FileContent` to `contentBytes` which will be returned in the end.

#### `AppendToFile(filename string, content []byte)`

- Get the `ShareNode` corresponding to the username and filename. Using information in `ShareNode` to retrive the `FileBody`. We will need information in `ShareNode` to update `FileBody` in _DataStore_ later.
- Create a new `FileContent` with randomly generated UUID. This UUID will be assigned to `LastContent` of `FileBody`, and the `LastContent` will be assigned to `PrevContent` in newly created `FileContent`.
- Finally, store both the new `FileContent` and updated `FileBody` to _DataStore_.

### File Sharing and Revocation

> File sharing is maintained by `ShareNode` tree structure. `ShareNode` will have different behaviors during create and accept invitation based on `ChildrenName`.
>
> All the file base keys will be regenerated and distributed along lockbox of the remaining `ShareNode`.

#### `CreateInvitation(filename string, recipientUsername string)`

- Retrieve `ShareNode` of sender using filename and sender name.
- If sender node is the root (`ChildrenName` is not `nil`)
  - Create a new ShareNode for the recipient. The UUID of this node is generated from `sendername + recipientname + filename`. `ChildrenName` will be `nil`, file base key will be copied from sender lockbox to newly created recipient lockbox. Recipient username will be appended to the `ChildrenName` array of sender.
  - A child specific base key will be generated based on the `SNBaseKey` of sender `ShareNode` and recipient username on fly so that different direct children `ShareNode` will have different base keys. Since we store the children username in `ChildrenName`, we can traverse direct children `ShareNode` later while revoking.

- If sender node is not the root
  - Simply use the current ShareNode as the recipient ShareNode since only the owner will be able to revoke the access to its direct recipients.

- Using the recipient ShareNode we created above, we will create a new `Invitation` which store the recipient `ShareNode` UUID and child specified `ShareNode` base key.
- Finally, this invitation is encrypted using the `RSAPublicKey` from recipient stored on _KeyStore_ and sign the encryed data using `DSSignKey` in `User` of recipient. This cyphertext and signature pair will be stored in `PublicEncData`. The UUID of `InvitationData` is generated from `"Invitation: " + sendername + filename + recipientname`. Note that while children `ShareNode` also generate UUID from these three string, they are using the different combination and extra string `"Invitation: "`, so that the hashed bytes will have no collision.
- `PublicEncData` is directly putting into _DataStore_ without protection.

#### `AcceptInvitation(senderUsername string, invitationPtr UUID, filename string)`

- Retrieve `InvitationData` after checking that the UUID exist and delete the `Invitation` in _DataStore_.
- Decrypt cyphertext using private key stored in recipient `User` after verifing the signature using digital signature verify key stored in _KeyStore_.
- Create the `FileHeader` whose UUID is generated from filename and username, store the `ShareNode` UUID in `Invitation` to newly created `FileHeader`.

#### `RevokeAccess(filename string, recipientUsername string)`

- Revocation will be done only on the `ShareNode` level. The only change after revocation is the `FileBaseKey` in the `Lockbox`. By store the updated key in lockbox, we can revoke recipients' access.
- Check if the `Invitation` is accpted and if the shared file exist. If so, we will delet corresponding `ShareNode` and `Invitation`.

- While deleting the child `ShareNode`, we first derive a new file base key and distribute it to all the child `ShareNode` other than the revoked one. Only the Lockbox is re-encrupted and stored at the same location.

## Helper Methods

```go
/* Store an object into the Datastore with given UUID */
func storeSymEncObject(dataId UUID, object interface{}, encKey []byte, macKey []byte) (err error)

/* Get an object from the Datastore */
func getSymEncObject(dataId UUID, encKey []byte, macKey []byte, object interface{}) (err error)

/* Store an object into the Datastore with given UUID */
func storePublicEncObject(dataId UUID, object interface{}, publicKey PKEEncKey, signKey DSSignKey) (err error)

/* Get an object from the Datastore */
func getPublicEncObject(dataId UUID, privateKey PKEDecKey, verifyKey DSVerifyKey, object interface{}) (err error)

/* Get Fileheader from username and filename */
func (userdata *User) getFileHeader(filename string, fileHeader interface{}) (err error)

/* Get ShareNode from username and filename */
func (userdata *User) getShareNode(filename string, shareNode interface{}) (err error)

/* Get file base key from ShareNode */
func (shareNode *ShareNode) getSNFileBaseKey() (fileBaseKey []byte, err error)

/* Get FileBody from username and filename*/
func (userdata *User) getFileBody(filename string, fileBody interface{}) (err error)

/* Get UUID from byte string */
func getUUIDFromString(bytes []byte) (uid UUID, err error)

/* Get base key from username + password to derive enc/mac key*/
func getUserBaseKey(username string, password []byte) (baseKey []byte)

/* Get FileHeader base key from user base key and filename*/
func getFileHeaderBaseKey(originalBaseKey []byte, filename string) (newBaseKey []byte, err error)

/* Get encryption & MAC key from base key */
func getKeyPairFromBase(baseKey []byte) (encKey []byte, macKey []byte, err error)

/* Get encryption & MAC key for the ShareNode of a given recipient */
func getChildBaseKey(baseKey []byte, recipientName string) (childBaseKey []byte, err error)

/* Get a new Lockbox base key */
func getLockboxBaseKey(baseKey []byte) (lockboxBaseKey []byte, err error)

/* Get next base key from current one */
func getNextBaseKey(originalBaseKey []byte) (newBaseKey []byte, err error)
```
