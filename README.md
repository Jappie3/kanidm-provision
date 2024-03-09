[About](#kanidm-provision) \| [Usage](#usage) \| [JSON Schema](#json-schema)

## 🦀 Kanidm Provisioning Tool

This is a tiny helper utility that uses kanidm's API to provision
users, groups and oauth2 systems. This tool is needed to allow declarative
provisioning of kanidm in NixOS, but can be used on any other system as well.
A optional patch for kanidm is provided that additionally allows you to provision oauth2 basic secrets.

Currently this supports the following operations which should suffice for basic SSO and OIDC needs.
PRs are of course welcome!

Category | Provisioning Feature | Supported
---|---|---
Groups | |
  | | Create/delete | ✅
  | | Members | ✅
  | | Unix attributes | ❌
Persons | |
  | | Create/delete | ✅
  | | Attributes (displayname, legalname, mail) | ✅
  | | Credentials | ❌
  | | SSH | ❌
  | | Unix attributes | ❌
  | | Radius | ❌
Oauth2 Resource Server | |
  | | Create/delete | ✅
  | | Attributes (origin url, origin landing, pkce enable, prefer short username) | ✅
  | | Basic secret | ✅ (Requires patch, [see below](#provisioning-oauth2-basic-secrets))
  | | Scope maps | ✅
  | | Supplementary scope maps | ✅
  | | Claim maps | ✅

## Usage

Build the utility simply by running:

```bash
> cargo build
```

Afterwards you can apply a state file to your kanidm instance by executing:

```bash
export KANIDM_PROVISION_IDM_ADMIN_TOKEN="your-idm-admin-token"
kanidm-provision --url 'https://auth.example.com' --state state.json
```

## Orphan removal

This tool automatically adds all created entities to a tracking group so
whenever something is removed from the state file in the future, the change
can be reflected in kanidm automatically.

To prevent this kind of orphan removal, you can to pass `--no-auto-remove`.
Removing for example a group from the state file will then not cause any
changes in kanidm, unless the state file explicitly specifies `present: false`.

This automatic tracking does not work for oauth2 claim maps, since claim maps
are not a separate entity in kanidm. To work around that, each oauth2 resource server
has a `removeOrphanedClaimMaps` option that will delete any claim maps on the resource
server that haven't been created by this tool. `--no-auto-remove` has no effect on that option.

## Provisioning oauth2 basic secrets

This tool is able to provision basic secrets if you build kanidm
with the patch provided in [./patches](./patches). This adds a new endpoint
that allows modifying the basic secret of any oauth2 resource server via a new API endpoint,
instead of only allowing to read the value generated by kanidm.

> \[!CAUTION\]
> You need to reset your kanidm database after applying this change, otherwise the access control
> profile will not be updated and any call to the new endpoint will fail!

## JSON Schema

This is the schema consumed by this application

```yaml
{
  # Specifies the provisioned groups
  "groups": {
    # One entry per group
    "group1": {
      # Optional. Defaults to true if not given.
      # Whether the group should be present or absent.
      "present": true,
      # The exhaustive list of group members.
	  "members": [
	    "person1",
	    "person2",
		"group1"
	  ]
    },
    # ...
  },
  # Specifies the provisioned persons
  "persons": {
    # One entry per person
    "person1": {
      # Optional. Defaults to true if not given.
      # Whether the person should be present or absent.
      "present": true,
      # Required.
      "displayName": "Person1",
      # Optional.
      "legalName": "Per Son",
      # Optional.
      "mailAddresses": [
        "person1@example.com"
        # ...
      ],
    },
    # ...
  },
  "systems": {
    "oauth2": {
      # One entry per oauth2 resource server
      "forgejo": {
        # Optional. Defaults to true if not given.
        # Whether the oauth2 resource server should be present or absent.
        "present": true,
        # Required.
        "displayName": "Forgejo",
        # Required. Must end with a '/'
        "originUrl": "https://git.example.com/",
        # Optional. Only works when using the patch. Do not specify otherwise!
        # Will set the basic secret to the contents of the given file. Whitespace will be trimmed from both ends.
        "basicSecretFile": "./secret1",
        # Optional.
        # Scope maps will map kanidm groups to returned oauth scopes.
        "scopeMaps": {
          # One entry per scope map.
          "group1": [
            "openid",
            "email",
            "profile"
          ]
        },
        # Optional.
        # Supplementary scope maps will map kanidm groups to additionally returned oauth scopes.
        "supplementaryScopeMaps": {
          # One entry per supplementary scope map.
          "group2": [
            "additional_scope"
          ]
        },
        # Optional. Defaults to true.
        # If true, any claim maps found on the resource server that are
        # not explicitly specified in here will be removed.
		"removeOrphanedClaimMaps": true,
        # Optional.
        # Claim maps will add a new claim with values depending on the
        # kanidm groups of the authenticating party.
        "claimMaps": {
          # One entry per claim, the key is the new claim name.
		  "groups": {
            # Required.
            # The strategy used to join multiple values. One of:
            #   - "ssv" (space separated: one two three)
            #   - "csv" (comma separated: one,two,three)
            #   - "array" (array notation: ["one", "two", "three"])
			"joinType": "array",
            # Assign values based on kanidm groups.
            # At least one entry is required.
			"claimsByGroup": {
			  "group1": [
			    "user"
			  ],
			  "group2": [
			    "user",
			    "important_user"
			  ],
			  "group3": [
			    "admin"
			  ]
              # ...
			}
		  }
          # ...
        }
      }
    }
  }
}
```

## License

Licensed under either of

- Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

at your option.
Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in this crate by you, as defined in the Apache-2.0 license,
shall be dual licensed as above, without any additional terms or conditions.
