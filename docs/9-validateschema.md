


install apicurio registry operator

create project `kafka-registry`

create kafka.yaml

create topic.yaml

oc get ingresses.config/cluster -o jsonpath={.spec.domain}

change host before run
create apicurio-registry.yaml

don't worry about poddistrubtionbudget violated


REGISTRY_URL="http://$(oc get route | grep 'my-registry-app' | awk '{print $2}')/apis/registry/v3"

curl -s "${REGISTRY_URL}/system/info" | jq

try to open web browser to webui

curl -X POST "${REGISTRY_URL}/groups" \
     -H "Content-Type: application/json" \
     -d '{
           "id": "app-events-group",
           "name": "app-events-group",
           "description": "schema for event"
         }'


Enforce a Compatibility Rule at the Registry (Server-Side Validation)

This is the layer that stops a producer from evolving the schema in an incompatible way — separate from the client-side structural check.

# Set a group-wide BACKWARD compatibility rule so future schema
# changes must remain backward-compatible with prior versions
curl -s -X PUT "${REGISTRY_URL}/groups/app-events-group/rules/COMPATIBILITY" \
  -H "Content-Type: application/json" \
  -d '{"config": "BACKWARD"}'