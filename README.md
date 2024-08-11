# aws-ecs-helm-templates
https://ranbook.cloud/posts/aws-ecs-helm-charts/

## Helm Commands
```bash
git clone https://github.com/imran9m/aws-ecs-helm-templates.git
# Let's create folder to save the yaml files generated through helm.
mkdir -p helm-charts-out
# Set target example app name, target cluster and environment
target_env=dev
target_cluster=default
service_name=sample-demo-ecs-service
# Below helm generates task definition yaml needed for AWS CLI to create the task definition
helm template aws-ecs-helm-templates \
    -s templates/taskdefinition.yaml \
    -f ./aws-ecs-helm-templates/values.yaml \
    --set environment=dev \
    > ./helm-charts-out/taskdefinition_out.yaml
# Let's execute AWS CLI ECS to create task definition and capture the revision which we will use later for service.
revision=$(aws ecs register-task-definition --cli-input-yaml file://helm-charts-out/taskdefinition_out.yaml --query 'taskDefinition.revision')

# Generate service specicification.
helm template aws-ecs-helm-templates \
    -s templates/service.yaml \
    -f ./aws-ecs-helm-templates/values.yaml \
    --set revision=$revision \
    --set environment=$target_env \
    > ./helm-charts-out/service_out.yaml
# Let's check if service already exists or not.
serviceName=$(aws ecs describe-services --cluster $target_cluster --services "$service_name-$target_env" --query 'services[0].serviceName')

# if service is null then let's create it or else update with revision from earlier command.
if [[ $serviceName == "null" ]]
then
  echo -e "creating the service!!\n"
  aws ecs create-service \
      --cluster $target_cluster \
      --service-name "$service_name-$target_env" \
      --cli-input-yaml file://helm-charts-out/service_out.yaml
else
  echo -e "updating the existing service!!\n"
  aws ecs update-service \
      --cluster $target_cluster \
      --service "$service_name-$target_env" \
      --force-new-deployment \
      --cli-input-yaml file://helm-charts-out/service_out.yaml
fi
```