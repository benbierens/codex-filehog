# Project: FileHog

## Use case
As a: Perfectly normal and regular computer user.

I want to: Store my folder with a lot of very important family photo and video files in a secure, durable, and privacy preserving manner.

So that I can: Reliably access them at a later time after my computer's hard drive has failed, while also not handing them over to a centralized potentially untrustworthy third party.

I want a tool that I can run on my folder that accomplishes this for me automatically.

## Codex
Codex is an open-source decentralized data storage engine with strong durability and privacy guarantees. For this project, we will use Codex's API to perform the data storage.
Docs: https://docs.codex.storage
API: https://api.codex.storage

## Terms
- CID = Content identifier, uniquely identified a file or dataset.
- TST / TSTWEI = A token without monitary value used as currency in the Codex testnet.
- Target folder = The folder which content is being saved.
- Output folder = A folder where the tool can create files to store its output and possible log files or crash reports.

## Constraints and Prerequisites
- The content of the files in the target folder does not change.
- Files are never deleted from the target folder.
- The files in the target folder are between 1MB and 1GB in size.
- One or more Codex REST API endpoints are provided. These Codex nodes are configured correctly to perform storage marketplace purchases and have been provisioned with the tokens required to do so.

## Requirements
### [1.0] Configuration
- [1.1] If the target folder and the output folder are idential, the tool crashes with a clear error message at startup
- [1.2] If any of the provided Codex API endpoints cannot be reached, the tool crashes with a clear error message at startup
- [1.3] The following aspects of the tool are easily configurable by a moderately experienced computer user:
  - The target folder
  - The output folder
  - Structure of the output: flattened or structured
  - Codex API endpoints being used
  - Details of the storage purchases: (from Codex API)
    - Price (per byte per second, default: 1000 TSTWEI)
    - Durability parameters (number of storage nodes, fault tolerance, default: 10/5)
    - Proof probability (default: 100)
    - Duration (default: 6 days)
    - Expiry (default: 1 hour)
    - Collateral requirement (per byte, default: 1 TSTWEI)
- [1.4] If the duration value is less than 1 day, the tool crashes with a clear error message at startup
- [1.5] If the expiry value is less than 15 minutes, the tool crashes with a clear error message at startup
- [1.6] If the expiry value is greater than the duration, the tool crashes with a clear error message at startup

### [2.0] Ingestion
- [2.1] The tool performs the following steps for each file in the target folder:
  - [2.2] If no storage purchase is currently active for the file, the tool will:
    - [2.3] Upload the file to a Codex node.
    - [2.4] Purchase storage for the file.
    - [2.5] Confirm that the purchase contract has been started.

- [2.6] For each file processed, the tool will save the following values to the output folder:
    - CID of original upload dataset
    - CID of storage contract dataset
    - Purchase ID
    - Timestamp of the creation of the purchase
    - Which Codex node was used for this file
    - Possible error information
  
- The tool will save the values in the output folder in accordance with the configured ([1.3]) structure:
  - [2.7] Flattened:
    - A single file is created or updated that contains the values for each file processed. In addition to values listed at [2.6], the relative path of the file in the target folder must be included.
  - [2.8] Structured:
    - A file is created or updated for each file processed. The name and relative path of this output file matches exactly the name and relative path of the file in the target folder, except a type extension is added. For example: ".json" in case of JSON files.

### [3.0] Purchasing
- [3.1] The tool will use the Codex API to create storage purchases in accordance with the configured ([1.3]) purchase values.
- [3.2] If purchasing fails because of an incorrect configuration value, then the tool crashes with a clear error message. (Codex API should indicate invalid values with a return code/error message.)

### [4.0] Monitoring
- [4.1] The tool will monitor the target folder for newly added files. These files will be included in its operation.
- [4.2] The tool will monitor the storage purchases it has created:
  - [4.3] When a storage purchase for a file fails, the tool will create a new purchase for it.
  - [4.4] When a storage purchase for a file will become finished within the next hour, the tool will create a new purchase for it.
- [4.5] Monitoring continues until explicit user action stops the tool.

### [5.0] Error handling
- [5.1] When disc operations fail, the tool crashes with a clear error message.
- [5.2] When network operations fail, the tool can attempt a retry no more than 3 times, then crashes with a clear error message.
- [5.3] When file uploads fail, the tool will store the error information in the output folder entry for that file and continue operating.
- [5.4] When the creation of a storage purchase fails due to lack of tokens, the tool crashes with a clear error message.
- [5.5] When the creation of a storage purchase fails for any other reason, or when a storage purchase fails to reach started state (it reaches cancelled or expired state instead), then the tool will store the error information in the output folder entry for that file and continue operating.

### [6.0] Other
- [6.1] When multiple Codex API endpoints are provided, the tool applies load-balancing to utilize all endpoints simultaneously.
- [6.2] The tool should support these operating systems: Ubuntu, macOS, and Windows
- [6.3] The tool should support operating in an environment without graphical shell
- [6.4] For debugging purposes, the tool should write logging information and crash reports to the output folder

### Good to know
- "CID of original upload dataset": Returned by the API POST request to `/data`
- "Purchase ID": Returned by the API POST request to `/storage/request/{cid}`
- "CID of storage contract dataset": Returned in the JSON structure under `request/content/cid` by the API GET request to `/storage/purchases/{id}`. This CID will be different from the one used to create the purchase.
- "A moderately experienced computer user": These users are comfortable operating a terminal and editing config files. They understand file system paths and can fix common setting issues. They can't necessarily write or understand code.

### Bonus goals
With the above requirements implemented, the tool should cover its intended use case. However, there are improvements that can be created to make using the tool easier. Here are some suggestions for additional features. If you come up with ideas of your own, you are invited and encouraged to contribute them!
- A method for notifying the user when:
  - All files are saved successfully
  - Errors occur
  - The tool crashes
- When a Codex node runs out of tokens (instead of crashing [5.4]):
  - Notify the user
  - Monitor the balance
  - Discontinue the use of the Codex node with insufficient tokens until the balance has been (sufficiently) increased
- The output folder ends up containing all the information necessary to retrieve the saved files. It would be ideal if this information could be saved as well. Find a way to upload and store the contents of the output folder that results in a single CID. Notify the user with this CID. Perhaps make this message machine-readable, so other applications can consume it and react.
- "Constraints and Prerequisites" defined a file size range limit. This is mostly due to the technical restrictions imposed by the current version of Codex. Find a way to support files of any size. Perhaps files that are too small can be combined or padded somehow. Perhaps files that are too large can be split somehow.
- It's possible that due to no fault of the tool, the purchase contracts are not being started. Perhaps there are not enough storage nodes available in the Codex network, resulting in storage contracts timing out and reaching "failed" state. The tool *should* upload all data as quickly as possible. But perhaps when it detects a fault of the Codex network of this kind, it should slow down, maybe attempting 1 purchase every 1 hour. Then, if it detects that 3 consecutive purchases start successfully, it could resume uploading and storing as fast as possible.
- Improvements to make it easy for users to set up, configure, and run the tool are always nice.
