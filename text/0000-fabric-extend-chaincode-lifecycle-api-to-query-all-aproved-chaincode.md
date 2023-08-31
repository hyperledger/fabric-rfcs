---
layout: default
title: Extend chaincode lifecycle API to query all approved chaincodes
nav_order: 3
---

- Feature Name: extend_chaincode_lifecycle_function_to_query_all_approved_chaincode
- Start Date: 2021-08-02
- RFC PR: (leave this empty)
- Fabric Component: fabric, fabric-protos
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

## Extend chaincode lifecycle API to query all approved chaincodes

Currently, Fabric peer CLI 'queryapproved' could query the details of an approved chaincode definitions, and this CLI requires a mandatory chaincode name parameter to query a specific chaincode definitions; however, in some cases we expect to query out all approved chaincode definitions.

So, here we propose to extend the 'queryapproved' function for querying the details of all the approved chaincode definition to Fabric peer CLI.

The following shows a basic design of this extension.

- There is no new command is involved.
- Command 'queryapproved' will be extended to support this function:
  - make parameter (-n|--name [chaincode_name]) optional
  - if parameter (-n|--name [chaincode_name]) is provided, keep current command behavior
  - if parameter (-n|--name [chaincode_name]) is not provided, query the details of all approved chaincode, plus additional chaincode name in details output.

*The example of query all approved chaincodes with JSON output format:*
```
$ peer lifecycle chaincode queryapproved --channelID <channelname> --peerAddresses peer0.org1.example.com:7051 --output json
{
	"chaincode_definitions": [
		{
			"name": "<ccname1>",
			"sequence": 1,
			"version": "1.0",
			"endorsement_plugin": "escc",
			"validation_plugin": "vscc",
			"validation_parameter": "EiAvQ2hhbm5lbC9BcHBsaWNhdGlvbi9FbmRvcnNlbWVudA==",
			"collections": {},
			"init_required": true,
			"source": {
				"Type": {
					"LocalPackage": {
						"package_id": "<ccname1>-1.0:e60b4fc692998844183e70ca6ae15bcd4632ef1b0e93193567a6669fb945d86d"
					}
				}
			}
		},
		{
			"name": "<ccname2>",
			"sequence": 1,
			"version": "1.0",
			"endorsement_plugin": "escc",
			"validation_plugin": "vscc",
			"validation_parameter": "EiAvQ2hhbm5lbC9BcHBsaWNhdGlvbi9FbmRvcnNlbWVudA==",
			"collections": {},
			"init_required": true,
			"source": {
				"Type": {
					"LocalPackage": {
						"package_id": "<ccname2>-1.0:23d718fa220eb599a412dfea13c18958b58fd0ffe4a42e2335b17c2c5fa102e9"
					}
				}
			}
		},
		{
			"name": "<ccname2>",
			"sequence": 2,
			"version": "2.0",
			"endorsement_plugin": "escc",
			"validation_plugin": "vscc",
			"validation_parameter": "EiAvQ2hhbm5lbC9BcHBsaWNhdGlvbi9FbmRvcnNlbWVudA==",
			"collections": {},
			"init_required": true,
			"source": {
				"Type": {
					"LocalPackage": {
						"package_id": "<ccname2>-2.0:23d718fa220eb599a412dfea13c18958b58fd0ffe4a42e2335b17c2c5fa102e9"
					}
				}
			}
		}
	]
}
```

# Motivation
[motivation]: #motivation

In some cases we expect to query out all approved chaincode definitions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The original 'queryapproved' implementation can be found by referring to a series of PRs, including this [PR](https://github.com/hyperledger/fabric-protos/pull/25).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The command reference of the original 'queryapproved' is [here](https://hyperledger-fabric.readthedocs.io/en/latest/commands/peerlifecycle.html#peer-lifecycle-chaincode-queryapproved).

# Testing
[testing]: #testing

- Enhance existing unit and integration tests on 'queryapproved' command.

# Dependencies
[dependencies]: #dependencies

- fabric-protos
	- This feature needs to extend lifecycle protos (e.g., Adding 'QueryApprovedChaincodeResults').
