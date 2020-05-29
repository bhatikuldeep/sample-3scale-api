# sample-3scale-api

oc process -f setup.yaml \
 -p DEVELOPER_ACCOUNT_ID="$SAAS_DEVELOPER_ACCOUNT_ID" \
           -p PRIVATE_BASE_URL="http://echo-api.3scale.net:80" \
           -p NAMESPACE="$TOOLBOX_NAMESPACE" |oc create -f -
