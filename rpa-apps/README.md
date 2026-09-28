# RPA scripts

## 1. Bot Script Architecture and Analysis

To illustrate script execution in production, let us analyze the anatomy of a modular WAL (Workplace Application Language) automation script [`MyBot1.wal`](MyBot1.wal).

### 1.1 Script Structure and Lifecycle

The script follows structured automation engineering standards with variable handling, subroutine decomposition, formatted timestamp generation, Windows Event logging, and central error management:

```
+-----------------------------------------------------------------------------+
|                             Bot1 Execution Flow                             |
|                                                                             |
|  [Main Entry]                                                               |
|     |                                                                       |
|     +--> Sub Init           (Validate required input / get Windows user)    |
|     +--> Sub BuildDateTime  (Extract date parts, pad, format timestamp)     |
|     +--> Sub LogRun         (Log start event to console & Windows Log)      |
|     +--> Sub Wait           (Perform deterministic pause / business delay)  |
|     +--> Sub LogFinish      (Log end event to console & Windows Log)        |
|     +--> Sub Cleanup        (Release resources and gracefully exit)         |
|                                                                             |
|  [Exception Handler: ErrorHandler] -> Stop execution on failure             |
+-----------------------------------------------------------------------------+
```

### 1.2 Key Technical Capabilities Demonstrated in Script

1. **Parameter Passing & Fallback**:
   ```wal
   defVar --name localInput --type String --value DefaultTestValue --parameter
   ...
   textNullOrEmpty --text "${localInput}" isNullOrEmpty=value
   if --left "${isNullOrEmpty}" --operator "Equal_To" --right True
       setVar --name "${localInput}" --value "No input found"
   endIf
   ```
   The script declares `localInput` as an external entry parameter marked `--required`. If called without explicit payload, fallback handling prevents null dereferencing.

2. **Date/Time Arithmetic & Zero-Padding**:
   Extracts distinct date/time integers (`Years`, `Months`, `Days`, `Hours`, `Minutes`, `Seconds`), applies `padText --paddingchar 0`, and constructs a normalized ISO-like string `YYYY-MM-DD HH:MM:SS`.

3. **Operating System Level Event Log Integration**:
   ```wal
   logMessage --message "Bot1 run at ${dateTimeOfRun} with param value [ ${localInput} ] using userId [ ${theUserId} ]" --type "Info" --logonwindows --eventid 987
   ```
   Emits structured diagnostic events directly into Windows Event Viewer using application-specific Event IDs (`987` for job initiation, `988` for job completion).


## 2. RPA demo configuration steps

...to be completed


## 3. RPA API examples using cUrl tool

```bash

#---------------------------------------
# ZEN Authentication
_TNS=cp4ba-rpa
_ROUTE_NAME="platform-id-provider"

# get pak admin username / password
ADMIN_USERNAME=$(oc get secret platform-auth-idp-credentials -n ${_TNS} -o jsonpath='{.data.admin_username}' | base64 -d)
ADMIN_PASSW=$(oc get secret platform-auth-idp-credentials -n ${_TNS} -o jsonpath='{.data.admin_password}' | base64 -d)

echo "${ADMIN_USERNAME} / ${ADMIN_PASSW}"

CONSOLE_HOST="https://"$(oc get route -n ${_TNS} ${_ROUTE_NAME} -o jsonpath="{.spec.host}")
PAK_HOST="https://"$(oc get route -n ${_TNS} cpd -o jsonpath="{.spec.host}")

echo "$PAK_HOST"

# get IAM access token
IAM_ACCESS_TK=$(curl -sk -X POST -H "Content-Type: application/x-www-form-urlencoded;charset=UTF-8" \
      -d "grant_type=password&username=${ADMIN_USERNAME}&password=${ADMIN_PASSW}&scope=openid" \
      ${CONSOLE_HOST}/idprovider/v1/auth/identitytoken | jq -r .access_token)
echo "${IAM_ACCESS_TK}"

ZEN_TK=$(curl -sk "${PAK_HOST}/v1/preauth/validateAuth" -H "username:${ADMIN_USERNAME}" -H "iam-token: ${IAM_ACCESS_TK}" | jq -r .accessToken)
echo "${ZEN_TK}"

#---------------------------------------
# API IAM

# get user info
_USER="vuxuser1"
_ACTION="usermgmt/v2/usermgmt/users"
RESPONSE=$(curl -sk -H "Authorization: Bearer ${ZEN_TK}" -H 'accept: application/json' -H 'Content-Type: application/json' -X GET "${PAK_HOST}/${_ACTION}?include_groups=true&offset=0&limit=25&search_str=${_USER}&include_users_count=false&includeAll=true")
echo ${RESPONSE} | jq .[].username | sed 's/"//g'
_ORIG_ROLES=$(echo ${RESPONSE} | jq .[].user_roles | sed 's/\[//g' | sed 's/\]//g' | sed 's/ //g' | sed '/^$/d')
echo ${_ORIG_ROLES}

# onboard user with rpa-automation-user role
_USER="vuxuser1"
_DATA='{"username":"'${_USER}'","displayName":"'${_USER}'","email":"'${_USER}'@vuxprod.net","user_roles":['${_ORIG_ROLES}',"rpa-automation-user"]}'
_ACTION="usermgmt/v1/user"
RESPONSE=$(curl -sk -H "Authorization: Bearer ${ZEN_TK}" -H 'accept: application/json' -H 'Content-Type: application/json' -d $_DATA -X PUT "${PAK_HOST}/${_ACTION}/${_USER}")
echo ${RESPONSE} | jq .

#---------------------------------------
# API RPA
# https://www.ibm.com/docs/en/rpa/30.0.x?topic=call-authenticating-rpa-api

_USER_NAME="cp4admin"
_USER_MAIL="cp4admin@vuxprod.net"
_USER_PWD="dem0s"

# Tenant by name
_TENANT_NAME="ibm"
_TENANT_ID=$(curl -sk -X GET ${PAK_HOST}/rpa/api/v2.0/account/tenant?username=${_USER_MAIL} | jq '.[] | select(.name == "'${_TENANT_NAME}'")' | jq .id | sed 's/"//g')
echo " ${_TENANT_NAME} / ${_TENANT_ID}"

# List of Regions
curl -sk -X GET ${PAK_HOST}/rpa/api/v2.0/configuration/regions | jq .

# API URL od on-premise region
curl -sk -X GET ${PAK_HOST}/rpa/api/v2.0/configuration/regions | jq '.[] | select(.name == "on-premise")' | jq .apiUrl | sed 's/"//g'

# User access tokens
_USER_IAM_TK=$(curl -ks -H "Content-Type: application/x-www-form-urlencoded;charset=UTF-8" -d "grant_type=password&username=$_USER_NAME&password=$_USER_PWD&scope=openid" ${CONSOLE_HOST}/idprovider/v1/auth/identitytoken | jq -r .access_token)
# echo ${_USER_IAM_TK}

_USER_ACCESS_TK=$(curl -ks -H "username:$_USER_NAME" -H "iam-token: ${_USER_IAM_TK}" ${CONSOLE_HOST}/v1/preauth/validateAuth | jq -r .accessToken)
# echo ${_USER_ACCESS_TK}

# User info
_RPA_USER_INFO=$(curl -ks --cookie "ibm-private-cloud-session=${_USER_ACCESS_TK}" ${CONSOLE_HOST}/rpa/api/zen-token-login)
echo ${_RPA_USER_INFO} | jq .

# RPA user tokens
OIDC_TK=$(echo ${_RPA_USER_INFO} | jq -r .oidcAccessToken)
_TENANT_ID=$(echo ${_RPA_USER_INFO} | jq -r .tenants[0].id)

_RPA_ACCESS_TK=$(curl -ks -H "Content-Type: application/x-www-form-urlencoded" -d grant_type=password --cookie "ibm-private-cloud-session=${_USER_ACCESS_TK}" -H "tenantId:${_TENANT_ID}" -H "oidc-access-token:${OIDC_TK}" ${CONSOLE_HOST}/rpa/api/v1.0/token | jq -r .access_token)
echo ${_RPA_ACCESS_TK}

# Workspace (Tenant)
_WORKSPACE=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET ${PAK_HOST}/rpa/api/v2.0/workspace)
_WORKSPACE_ID=$(echo "${_WORKSPACE}" | jq .[].id | sed 's/"//g')
_WORKSPACE_NAME=$(echo "${_WORKSPACE}" | jq .[].name | sed 's/"//g')
echo "${_WORKSPACE_NAME} - ${_WORKSPACE_ID}"

# Workspace info
_WORKSPACE=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET ${PAK_HOST}/rpa/api/v2.0/tenant/${_WORKSPACE_ID})
_WORKSPACE_ID=$(echo "${_WORKSPACE}" | jq .id | sed 's/"//g')
echo ${_WORKSPACE} | jq .

# process list
curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/process?singleStepProcessesOnly=false&OrderBy=name&Asc=true" | jq .

# projects list
_PROJECTS=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/projects")
_PRJ_ID=$(echo ${_PROJECTS} | jq .[0].id | sed 's/"//g')
_PRJ_UNIQUE_ID=$(echo ${_PROJECTS} | jq .[0].uniqueId | sed 's/"//g')

echo "${_PRJ_UNIQUE_ID} / ${_PRJ_ID}"

# process list
curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/process" | jq .


# OpenAPI document
curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/projects/${_PRJ_ID}/openapi" | yq .

curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/projects/${_PRJ_UNIQUE_ID}/openapi" | yq .


# Bots list

curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/bots" | jq .


# Get Bot info
_BOT_NAME="MyBot1"
_BOT_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/bots" | jq '.[] | select(.name == "'${_BOT_NAME}'")')
echo ${_BOT_INFO}
_BOT_ID=$(echo ${_BOT_INFO} | jq .id | sed 's/"//g')
_BOT_PRJ_ID=$(echo ${_BOT_INFO} | jq .project.id | sed 's/"//g')
echo "Bot [${_BOT_NAME}] id[${_BOT_ID}] prjId[${_BOT_PRJ_ID}]"

_BOT_NAME="MyBot2"
_BOT_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/bots" | jq '.[] | select(.name == "'${_BOT_NAME}'")')
echo ${_BOT_INFO}
_BOT_ID=$(echo ${_BOT_INFO} | jq .id | sed 's/"//g')
_BOT_PRJ_ID=$(echo ${_BOT_INFO} | jq .project.id | sed 's/"//g')
echo "Bot [${_BOT_NAME}] id[${_BOT_ID}] prjId[${_BOT_PRJ_ID}]"

# Run bot (no input params, error if param required)

_BOT_NAME="MyBot1"
_BOT_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/bots" | jq '.[] | select(.name == "'${_BOT_NAME}'")')
echo ${_BOT_INFO}
_BOT_ID=$(echo ${_BOT_INFO} | jq .id | sed 's/"//g')
_BOT_PRJ_ID=$(echo ${_BOT_INFO} | jq .project.id | sed 's/"//g')

_BOT_RUN_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X POST "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/projects/${_BOT_PRJ_ID}/bots/${_BOT_ID}")

_ERR_MSG=$(echo ${_BOT_RUN_INFO} | jq .errorMessage | sed 's/"//g')
if [[ ! -z "${_ERR_MSG}" && "${_ERR_MSG}" != "null" ]]; then
  echo "ERROR: ${_ERR_MSG}"
else
  _BOT_RUN_JOBID=$(echo ${_BOT_RUN_INFO} | jq .jobId | sed 's/"//g')
  _BOT_RUN_PRJ=$(echo ${_BOT_RUN_INFO} | jq .project | sed 's/"//g')
  _BOT_RUN_NAME=$(echo ${_BOT_RUN_INFO} | jq .botName | sed 's/"//g')
  _BOT_RUN_STATUS=$(echo ${_BOT_RUN_INFO} | jq .status | sed 's/"//g')
  _BOT_RUN_STATUS_NAME=$(echo ${_BOT_RUN_INFO} | jq .statusName | sed 's/"//g')
  echo "Bot [${_BOT_RUN_NAME}] job[${_BOT_RUN_JOBID}] status[${_BOT_RUN_STATUS_NAME}]"
fi



# Run bot 1 (with input params)

_BOT_NAME="MyBot1"
_BOT_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/bots" | jq '.[] | select(.name == "'${_BOT_NAME}'")')
echo ${_BOT_INFO}
_BOT_ID=$(echo ${_BOT_INFO} | jq .id | sed 's/"//g')
_BOT_PRJ_ID=$(echo ${_BOT_INFO} | jq .project.id | sed 's/"//g')

_BOT_INPUT_DATA='{"localInput": "Hello from curl !"}'

_BOT_RUN_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -H 'Content-Type: application/json' -d "${_BOT_INPUT_DATA}" -X POST "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/projects/${_BOT_PRJ_ID}/bots/${_BOT_ID}")
_ERR_MSG=$(echo ${_BOT_RUN_INFO} | jq .errorMessage | sed 's/"//g')
if [[ ! -z "${_ERR_MSG}" && "${_ERR_MSG}" != "null" ]]; then
  echo "ERROR: ${_ERR_MSG}"
else
  _BOT_RUN_JOBID=$(echo ${_BOT_RUN_INFO} | jq .jobId | sed 's/"//g')
  _BOT_RUN_PRJ=$(echo ${_BOT_RUN_INFO} | jq .project | sed 's/"//g')
  _BOT_RUN_NAME=$(echo ${_BOT_RUN_INFO} | jq .botName | sed 's/"//g')
  _BOT_RUN_STATUS=$(echo ${_BOT_RUN_INFO} | jq .status | sed 's/"//g')
  _BOT_RUN_STATUS_NAME=$(echo ${_BOT_RUN_INFO} | jq .statusName | sed 's/"//g')
  echo "Bot [${_BOT_RUN_NAME}] job[${_BOT_RUN_JOBID}] status[${_BOT_RUN_STATUS_NAME}]"
fi

# Run bot 2 (with input params)

_BOT_NAME="MyBot2"
_BOT_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -X GET "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/bots" | jq '.[] | select(.name == "'${_BOT_NAME}'")')
echo ${_BOT_INFO}
_BOT_ID=$(echo ${_BOT_INFO} | jq .id | sed 's/"//g')
_BOT_PRJ_ID=$(echo ${_BOT_INFO} | jq .project.id | sed 's/"//g')

_BOT_INPUT_DATA='{"localInput": "Hello from curl !"}'

_BOT_RUN_INFO=$(curl -sk -H "Authorization: Bearer ${_RPA_ACCESS_TK}" -H 'Content-Type: application/json' -d "${_BOT_INPUT_DATA}" -X POST "${PAK_HOST}/rpa/api/v2.0/workspace/${_WORKSPACE_ID}/projects/${_BOT_PRJ_ID}/bots/${_BOT_ID}")
_ERR_MSG=$(echo ${_BOT_RUN_INFO} | jq .errorMessage | sed 's/"//g')
if [[ ! -z "${_ERR_MSG}" && "${_ERR_MSG}" != "null" ]]; then
  echo "ERROR: ${_ERR_MSG}"
else
  _BOT_RUN_JOBID=$(echo ${_BOT_RUN_INFO} | jq .jobId | sed 's/"//g')
  _BOT_RUN_PRJ=$(echo ${_BOT_RUN_INFO} | jq .project | sed 's/"//g')
  _BOT_RUN_NAME=$(echo ${_BOT_RUN_INFO} | jq .botName | sed 's/"//g')
  _BOT_RUN_STATUS=$(echo ${_BOT_RUN_INFO} | jq .status | sed 's/"//g')
  _BOT_RUN_STATUS_NAME=$(echo ${_BOT_RUN_INFO} | jq .statusName | sed 's/"//g')
  echo "Bot [${_BOT_RUN_NAME}] job[${_BOT_RUN_JOBID}] status[${_BOT_RUN_STATUS_NAME}]"
fi

```


