all:
	kubectl create secret docker-registry {{ cookiecutter.project_name }}-registry -n {{ cookiecutter.namespace }} --dry-run=client --from-file=.dockerconfigjson={{ cookiecutter.docker_config }} -o yaml > {{ cookiecutter.project_name }}-registry.yaml
