SELDON_CORE_DIR=..
VERSION := $(shell cat ../version.txt)

.PHONY: get_apis
get_apis:
	# Protobuf
	cp ${SELDON_CORE_DIR}/proto/prediction.proto seldon_core/proto/
	$(MAKE) -C ../proto/tensorflow/ create_protos
	cp -r $(SELDON_CORE_DIR)/proto/tensorflow/tensorflow \
		seldon_core/proto/
	$(MAKE) -C ../proto/tensorflow clean

	# OpenAPI
	cp $(SELDON_CORE_DIR)/openapi/wrapper.oas3.json seldon_core/openapi/seldon.json

.PHONY: build_apis
build_apis: get_apis
	# Protobuf
	cd seldon_core && poetry run python \
		-m grpc.tools.protoc \
		-I./ \
		-I./proto/ \
		--python_out=./ \
		--grpc_python_out=./ \
		--mypy_out=./ \
		./proto/prediction.proto
	## We build TF's protobufs as well in case the tensorflow package
	## is not found
	cd seldon_core/proto && poetry run python \
		-m grpc.tools.protoc \
		-I./ \
		-I../ \
		--python_out=./ \
		./tensorflow/core/framework/*.proto
	sed -i "s/from tensorflow/from seldon_core.proto.tensorflow/" seldon_core/proto/*.py
	sed -i "s/from tensorflow.core.framework/from ./" \
		$(addprefix seldon_core/proto/tensorflow/core/framework/, \
			resource_handle_pb2.py tensor_pb2.py tensor_shape_pb2.py types_pb2.py)
	sed -i "s/from proto/from seldon_core.proto/g" seldon_core/proto/prediction_pb2_grpc.py

.PHONY: update_version
update_version:
	sed -i 's/__version__ = .*/__version__ = "$(VERSION)"/g' seldon_core/version.py

.PHONY: update_package
update_package: get_apis build_apis update_version

.PHONY: install
install:
	poetry install --only main

.PHONY: install_dev
install_dev:
	poetry install

.PHONY: licenses
licenses:
	poetry run tox --recreate -e licenses
	cut -d, -f1,3 ./licenses/license_info.csv \
		> ./licenses/license_info.no_versions.csv

.PHONY: uninstall
uninstall:
	poetry env remove --all

.PHONY: test
test:
	poetry run tox

.PHONY: type_check
type_check:
	poetry run mypy --ignore-missing-imports --no-strict-optional seldon_core

.PHONY: build_pypi
build_pypi:
	poetry build

.PHONY: build_conda
build_conda:
	mkdir -p conda-bld
	poetry run conda build conda.recipe/meta.yaml -c conda-forge --output-folder conda-bld

.PHONY: push_pypi_test
push_pypi_test:
	poetry publish --repository testpypi

.PHONY: push_pypi
push_pypi:
	poetry publish

.PHONY: push_conda
push_conda:
	@echo "1st push: need to submit a pull-request to https://github.com/conda-forge/staged-recipes"
	@echo "Subsequent pushes: need to for the python-seldon-core feedstock"
	@echo "Alternatively use anaconda upload to publish on own channel"

setup_linter:
	@echo "Linters are now managed by Poetry. Run 'poetry install' to set up."

fmt:
	poetry run black \
		./ ../testing/scripts ../operator/helm ../operator/hack \
		--exclude "(proto|seldon_core/proto/|.eggs|.tox)"
	poetry run isort \
		./ ../testing/scripts ../operator/helm ../operator/hack \
		--skip proto --skip seldon_core/proto --skip .eggs --skip .tox
	poetry run nbqa black \
		../ --nbqa-exclude "(proto|seldon_core/proto/|.eggs|.tox|_README\.ipynb)"
	poetry run nbqa isort \
		../ --profile black --nbqa-exclude "(proto|seldon_core/proto/|.eggs|.tox|_README\.ipynb)"

lint:
	poetry run black \
		--check ./ ../testing/scripts \
		--exclude "(proto|seldon_core/proto/|.eggs|.tox)"
	poetry run isort \
		--check-only ./ ../testing/scripts \
		--skip proto --skip seldon_core/proto --skip .eggs --skip .tox
	poetry run flake8
	poetry run nbqa black \
		../ --check --nbqa-exclude "(proto|seldon_core/proto/|.eggs|.tox|_README\.ipynb)"
	poetry run nbqa isort \
		../ --check-only --profile black --nbqa-exclude "(proto|seldon_core/proto/|.eggs|.tox|_README\.ipynb)"
	# Check if licenses have changed
	git \
		--no-pager diff \
		--exit-code \
		./licenses/license_info.no_versions.csv

.PHONY: clean
clean:
	rm -rf seldon_core.egg-info
	rm -rf seldon_core/tensorflow
	rm -rf .eggs
	rm -rf .pytest_cache
	rm -rf dist
	rm -rf conda-bld
	rm -rf conda_pkg_path.txt
	rm -rf .empty
	rm -rf build
