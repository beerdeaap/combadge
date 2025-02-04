---
hide:
  - navigation
  - toc
---

# Overview

Combadge generates a service client implementation from a user service interface
declared by a [protocol](https://peps.python.org/pep-0544/) class or an abstract base class.

[![Checks](https://img.shields.io/github/checks-status/kpn/combadge/main?logo=github)](https://github.com/kpn/combadge/actions/workflows/check.yaml)
[![Coverage](https://codecov.io/gh/kpn/combadge/branch/main/graph/badge.svg?token=ZAqYAaTXwE)](https://codecov.io/gh/kpn/combadge)
![Code style](https://img.shields.io/badge/code%20style-black-000000.svg)
[![Python Version](https://img.shields.io/pypi/pyversions/combadge?logo=python&logoColor=yellow)](https://pypi.org/project/combadge/)
[![License](https://img.shields.io/github/license/kpn/combadge)](LICENSE)

!!! warning "This package is in active development"

    The documentation is not good and complete as it should be, and the implementation may change drastically.

## Features

- Request and response models based on [**pydantic**](https://docs.pydantic.dev/)
- Declarative services using [`Protocol`](https://peps.python.org/pep-0544/)
- Exception classes automatically derived from error models
- Built-in backends:
    - [HTTPX](https://www.python-httpx.org/) – sync and async
    - [Zeep](https://docs.python-zeep.org/en/master/) – sync and async

## Sneak peek

=== "With HTTPX"

    ```python title="quickstart_httpx.py"
    from http import HTTPStatus
    from typing import List

    from httpx import Client
    from pydantic import BaseModel, Field, validate_arguments
    from typing_extensions import Annotated, Protocol

    from combadge.core.binder import bind
    from combadge.core.markers.method import wrap_with
    from combadge.support.http.aliases import StatusCode
    from combadge.support.http.markers import QueryParam, http_method, path
    from combadge.support.httpx.backends.sync import HttpxBackend


    # 1️⃣ Declare the response models:
    class CurrentCondition(BaseModel):
        humidity: int
        temperature: Annotated[float, Field(alias="temp_C")]


    class Weather(BaseModel):
        status: StatusCode[HTTPStatus]
        current: Annotated[List[CurrentCondition], Field(alias="current_condition")]


    # 2️⃣ Declare the protocol:
    class SupportsWttrIn(Protocol):
        @http_method("GET")
        @path("/{in_}")
        @wrap_with(validate_arguments)
        def get_weather(
            self,
            *,
            in_: Annotated[str, Field(min_length=1)],
            format_: Annotated[str, Field(min_length=1), QueryParam("format")] = "j1",
        ) -> Weather:
            raise NotImplementedError


    # 3️⃣ Bind the service:
    service = HttpxBackend(Client(base_url="https://wttr.in"))[SupportsWttrIn]

    # 🚀 Call the service:
    response = service.get_weather(in_="amsterdam")
    assert response.status == HTTPStatus.OK
    assert response.current[0].humidity == 71
    assert response.current[0].temperature == 8.0
    ```

=== "With Zeep"

    ```python title="quickstart_zeep.py"

    from typing import Literal, Union

    import zeep
    from pydantic import BaseModel, Field
    from pytest import raises
    from typing_extensions import Annotated, Protocol

    from combadge.core.interfaces import SupportsService
    from combadge.core.response import ErrorResponse, SuccessfulResponse
    from combadge.support.soap.markers import Body, operation_name
    from combadge.support.zeep.backends.sync import ZeepBackend


    # 1️⃣ Declare the request model:
    class NumberToWordsRequest(BaseModel, allow_population_by_field_name=True):
        number: Annotated[int, Field(alias="ubiNum")]


    # 2️⃣ Declare the response model:
    class NumberToWordsResponse(SuccessfulResponse):
        __root__: str


    # 3️⃣ Optionally, declare the error response models:
    class NumberTooLargeResponse(ErrorResponse):
        __root__: Literal["number too large"]


    # 4️⃣ Declare the interface:
    class SupportsNumberConversion(SupportsService, Protocol):
        @operation_name("NumberToWords")
        def number_to_words(self, request: Body[NumberToWordsRequest]) -> Union[NumberTooLargeResponse, NumberToWordsResponse]:
            ...


    # 5️⃣ Bind the service:
    client = zeep.Client(wsdl="tests/integration/wsdl/NumberConversion.wsdl")
    service = ZeepBackend(client.service)[SupportsNumberConversion]

    # 🚀 Call the service:
    response = service.number_to_words(NumberToWordsRequest(number=42))
    assert response.unwrap().__root__ == "forty two "

    # ☢️ Error classes are automatically derived for error models:
    response = service.number_to_words(NumberToWordsRequest(number=-1))
    with raises(NumberTooLargeResponse.Error):
        response.raise_for_result()
    ```
