# HTTP protocol for AI chat apps (Version 2024-01-28)

## Rationale
The Chat Protocol for AI applications featuring chat-like solutions and tools, is an effort to standarize API contracts across solutions and languages,
to make them compatible and interoperable. 

By agreeing on a common protocol, developers can swap components. For example, using the Python backend [azure-search-openai-demo](https://github.com/Azure-Samples/azure-search-openai-demo)
with the Web Components frontend from [azure-search-openai-javascript](https://github.com/Azure-Samples/azure-search-openai-javascript).

This protocol is inspired by the [OpenAI ChatCompletion API](https://platform.openai.com/docs/guides/text-generation/chat-completions-api),
but contains additional fields required for a chat application.

Table of contents:

* [HTTP requests to AI chat app endpoints](#http-requests-to-ai-chat-app-endpoints)
    * [Recommended request context](#recommended-request-context)
* [HTTP responses from RAG chat app endpoints](#http-responses-from-rag-chat-app-endpoints)
    * [Non-streaming response](#non-streaming-response)
        * [Successful response](#successful-response)
        * [Error response](#error-response)
    * [Streaming response](#streaming-response)
        * [Successful response](#successful-response-1)
        * [Error response](#error-response-1)
    * [Answer formatting](#answer-formatting)
    * [Recommended response context](#recommended-response-context)

## HTTP requests to the AI chat app endpoints

An HTTP request should always be a POST request, with the following headers:

* `Content-Type: application/json`
* `Authorization: Bearer <ID token>`: _Optional._ For applications that require authentication.

The body of the request can contain these properties, in JSON format:

* `"messages"`: A list of messages, each containing "content" and "role", where "role" may be "assistant" or "user". A single-turn chat app may only contain 1 message, while a multi-turn chat app may contain multiple messages.
* `"stream"`: A boolean indicating whether the response should be streamed or not.
* `"context"`: _Optional_. An object containing any additional context about the request, such as the temperature to use for the LLM. Each application may define its own context properties. See [recommended request context properties](#request-context).
* `"session_state"`: _Optional._ An object containing the "memory" for the chat app, such as a user ID.

### Usage example
The example belows represents a valid and compliant request body to the chat app endpoints:

```json
{
    "messages": [
        {
            "content": "What is included in my Northwind Health Plus plan that is not in standard?",
            "role": "user"
        }
    ],
    "stream": false,
    "context": {},
    "session_state": null
}
```

### Recommended request context

The request context object can contain any properties. However, here are some recommended common properties:

* `"overrides"`: An object containing settings for the chat application. 
    * `"temperature"`: The temperature to use for the LLM.
    * `"top"`: The number of results to return from the search engine.
    * `"retrieval_mode"`: The mode to use for the search engine. Can be "hybrid", "vectors", or "text".
    * `"semantic_ranker"`: _Specific to Azure AI Search_. Whether to use the semantic ranker for the search engine. 
    * `"semantic_captions"`: _Specific to Azure AI Search_. Whether to use semantic captions for the search engine.
    * `"suggest_followup_questions"`: Whether to suggest follow-up questions for the chat app.
    * `"use_oid_security_filter"`: Whether to use the OID security filter for the search engine.
    * `"use_groups_security_filter"`: Whether to use the groups security filter for the search engine.
    * `"vector_fields"`: A list of fields to search for the vector search engine.
    * `"use_gpt4v"`: Whether to use a GPT-4V approach.
    * `"gpt4v_input"`: The input type to use for a GPT-4V approach. Can be "text", "textAndImages", or "images".


Example of the overrides object:

```json
"overrides": {
    "top": 3,
    "retrieval_mode": "text",
    "semantic_ranker": false,
    "semantic_captions": false,
    "suggest_followup_questions": false,
    "use_oid_security_filter": false,
    "use_groups_security_filter": false,
    "vector_fields": ["embedding"],
    "use_gpt4v": false,
    "gpt4v_input": "textAndImages"
}
```


## HTTP responses from RAG chat app endpoints

The HTTP response should either be JSON for a non-streaming response, or [newline-delimited JSON](https://ndjson.org/home/) ("NDJSON"/"jsonlines") for a streaming response.

### Non-streaming response

This response format is based off the
[OpenAI chat completion object](https://platform.openai.com/docs/api-reference/chat/object),
with additional properties needed to properly display sources and citations.

The response should contain this header:

* `Content-Type: application/json`

#### Successful response

A successful response should have a status code of 200, and the body should contain a JSON object with the following properties:

* `"choices"`: A list of responses from the LLM, typically containing only 1 response as our app sets `n=1` when requesting a completion. Each response contains:
   * `"message"`: An object containing the actual content of the response.  See [Answer formatting](#answer-formatting). _Comes from the OpenAI response._
   * `"finish_reason"`: A string representing the finish state of the response. _Comes from the OpenAI response._
   * `"index"`: A number indicating which response this is (0 in the case of 1 response given). _Comes from the OpenAI response._
   * `"content_filter_results"`: An object from the Azure Content Safety filter. _Same as the OpenAI response, but will only be returned when using Azure OpenAI, not openai.com OpenAI._
   * `"context"`: _Optional_. An object containing additional details needed for the chat app. Each application can define its own properties. See [recommended response context properties](#response-context).
   * `"session_state"`: _Optional_. An object containing the "memory" for the chat app, such as a user ID. 
* `"created"`: The Unix timestamp (in seconds) of when the chat completion was created. _Comes from the OpenAI response._
* `"id"`: A unique identifier for the chat completion. _Comes from the OpenAI response._
* `"model"`: The model used for the completion, such as "gpt-35-turbo". _Comes from the OpenAI response._
* `"object"`: The object type, always "chat.completion". _Comes from the OpenAI response._
* `"prompt_filter_results"`: An object from the Azure Content Safety filter. _Same as the OpenAI response, but will only be returned when using Azure OpenAI, not openai.com OpenAI._
* `"system_fingerprint"`: Represents the backend configuration that the model runs with. _Same as the OpenAI response, but will only be returned when using Azure OpenAI, not openai.com OpenAI._
* `"usage"`: Usage statistics for the completion request. _Comes from the OpenAI response._


Here's an example JSON response:

```json
{
    "choices": [
        {
            "content_filter_results": {
                "hate": {
                    "filtered": false,
                    "severity": "safe"
                },
                "self_harm": {
                    "filtered": false,
                    "severity": "safe"
                },
                "sexual": {
                    "filtered": false,
                    "severity": "safe"
                },
                "violence": {
                    "filtered": false,
                    "severity": "safe"
                }
            },
            "context": {
                "data_points": {
                    "text": [
                        "Northwind_Standard_Benefits_Details.pdf#page=91:    Tips for Avoiding Intentionally False Or Misleading Statements:   When it comes to understanding a health plan, it is important to be aware of any  intentiona lly false or misleading statements that the plan provider may make...(truncated)",
                        "Northwind_Standard_Benefits_Details.pdf#page=91:  It is important to  research the providers and services offered in the Northwind Standard plan in order to  determine if the providers and services offered are sufficient for the employee's needs...(truncated)",
                        "Northwind_Standard_Benefits_Details.pdf#page=17:  Employees should keep track of their claims and follow up with  Northwind Health if a claim is not processed in a timely manner...(truncated)"
                    ]
                },
                "thoughts": [
                    {
                        "description": "What is included in my Northwind Health Plus plan that is not in standard?",
                        "props": null,
                        "title": "Original user query"
                    },
                    {
                        "description": "Northwind Health Plus plan coverage details compared to standard plan",
                        "props": {
                            "has_vector": false,
                            "use_semantic_captions": false
                        },
                        "title": "Generated search query"
                    },
                    {
                        "description": [
                            {
                                "captions": [],
                                "category": null,
                                "content": "  \nTips for Avoiding Intentionally False Or Misleading Statements:  \nWhen it comes to understanding a health plan, it is important to be aware of any \nintentiona lly false or misleading statements that the plan provider may make...(truncated)",
                                "embedding": null,
                                "groups": [],
                                "id": "file-Northwind_Standard_Benefits_Details_pdf-4E6F72746877696E645F5374616E646172645F42656E65666974735F44657461696C732E706466-page-233",
                                "imageEmbedding": null,
                                "oids": [],
                                "sourcefile": "Northwind_Standard_Benefits_Details.pdf",
                                "sourcepage": "Northwind_Standard_Benefits_Details.pdf#page=91"
                            },
                            {
                                "captions": [],
                                "category": null,
                                "content": " It is important to \nresearch the providers and services offered in the Northwind Standard plan i n order to \ndetermine if the providers and services offered are sufficient for the employee's needs...(truncated)",
                                "embedding": null,
                                "groups": [],
                                "id": "file-Northwind_Standard_Benefits_Details_pdf-4E6F72746877696E645F5374616E646172645F42656E65666974735F44657461696C732E706466-page-232",
                                "imageEmbedding": null,
                                "oids": [],
                                "sourcefile": "Northwind_Standard_Benefits_Details.pdf",
                                "sourcepage": "Northwind_Standard_Benefits_Details.pdf#page=91"
                            },
                            {
                                "captions": [],
                                "category": null,
                                "content": " Employees should keep track of their claims and follow up with \nNorthwind Health if a claim is not processed in a timely manner...(truncated)",
                                "embedding": null,
                                "groups": [],
                                "id": "file-Northwind_Standard_Benefits_Details_pdf-4E6F72746877696E645F5374616E646172645F42656E65666974735F44657461696C732E706466-page-41",
                                "imageEmbedding": null,
                                "oids": [],
                                "sourcefile": "Northwind_Standard_Benefits_Details.pdf",
                                "sourcepage": "Northwind_Standard_Benefits_Details.pdf#page=17"
                            }
                        ],
                        "props": null,
                        "title": "Results"
                    },
                    {
                        "description": [
                            "{'role': 'system', 'content': \"Assistant helps the company employees with their healthcare plan questions, and questions about the employee handbook. Be brief in your answers.\n        Answer ONLY with the facts listed in the list of sources below. If there isn't enough information below, say you don't know. Do not generate answers that don't use the sources below. If asking a clarifying question to the user would help, ask the question.\n        For tabular information return it as an html table. Do not return markdown format. If the question is not in English, answer in the language used in the question.\n        Each source has a name followed by colon and the actual information, always include the source name for each fact you use in the response. Use square brackets to reference the source, for example [info1.txt]. Don't combine sources, list each source separately, for example [info1.txt][info2.pdf].\n        \n        \n        \"}",
                            "{'role': 'user', 'content': \"What is included in my Northwind Health Plus plan that is not in standard?\n\nSources:\nNorthwind_Standard_Benefits_Details.pdf#page=91:    Tips for Avoiding Intentionally False Or Misleading Statements:   When it comes to understanding a health plan, it is important to be aware of any  intentiona lly false or misleading statements that the plan provider may make. To avoid  being misled, employees should follow the following tips:(truncated)
                            \nNorthwind_Standard_Benefits_Details.pdf#page=91:  It is important to  research the providers and services offered in the Northwind Standard plan in order to  determine if the providers and services offered are sufficient for the employee's needs.   In addition, Northwind Health may make claims that their plan offers low or no cost  prescription drugs..(truncated)\"}"
                        ],
                        "props": null,
                        "title": "Prompt"
                    }
                ]
            },
            "finish_reason": "stop",
            "index": 0,
            "message": {
                "content": "There is no specific information provided about what is included in the Northwind Health Plus plan that is not in the standard plan. It is recommended to read the plan details carefully and ask questions to understand the specific benefits of the Northwind Health Plus plan [Northwind_Standard_Benefits_Details.pdf#page=91].",
                "function_call": null,
                "role": "assistant",
                "tool_calls": null
            },
            "session_state": null
        }
    ],
    "created": 1706301586,
    "id": "chatcmpl-8lNHGormHX5fhozITxASIufDZno9D",
    "model": "gpt-35-turbo",
    "object": "chat.completion",
    "prompt_filter_results": [
        {
            "content_filter_results": {
                "hate": {
                    "filtered": false,
                    "severity": "safe"
                },
                "self_harm": {
                    "filtered": false,
                    "severity": "safe"
                },
                "sexual": {
                    "filtered": false,
                    "severity": "safe"
                },
                "violence": {
                    "filtered": false,
                    "severity": "safe"
                }
            },
            "prompt_index": 0
        }
    ],
    "system_fingerprint": null,
    "usage": {
        "completion_tokens": 64,
        "prompt_tokens": 967,
        "total_tokens": 1031
    }
}
```

#### Error response

An error response should have a status code of 400 or 500, and the body should contain a JSON object with the following properties:

* `"error"`: A string describing the error.

Here's an example JSON response for a 500-level error:

```json
{
    "error": "The app encountered an error processing your request.\nIf you are an administrator of the app, view the full error in the logs."
}
```

Here's an example JSON response for a 400-level error:

```json
{
    "error": "Your message contains content that was flagged by the OpenAI content filter."
}
```


### Streaming response

This response format is based off the
[OpenAI chat completion chunk object](https://platform.openai.com/docs/api-reference/chat/streaming),
with additional properties needed to properly display sources and citations.

The response should contain these headers:

* `Content-Type: application/json-lines`
* `Transfer-Encoding: chunked`

#### Successful response

A successful response should have a status code of 200.
The body of the response should contain a sequence of JSON objects, each representing a chunk of the response.
The first chunk contains a choice with the `context` property, since that is available before the answer, and subsequent chunks contain parts of the answer to the question.

Each JSON object should contain the following properties:

* `"choices"`: A list of responses from the LLM, typically containing only 1 response as our app sets `n=1` when requesting a completion. Each response contains:
   * `"delta"`: An object containing the actual content of the response, a token at a time. See [Answer formatting](#answer-formatting). _Comes from the OpenAI response._
   * `"finish_reason"`: A string representing the finish state of the response. _Comes from the OpenAI response._
   * `"index"`: A number indicating which response this is (0 in the case of 1 response given). _Comes from the OpenAI response._
   * `"content_filter_results"`: An object from the Azure Content Safety filter. _Comes from the OpenAI response, but will only be returned when using Azure OpenAI, not openai.com OpenAI._
   * `"context"`: _Optional_. An object containing additional details needed for the chat app. Each application can define its own properties. See [recommended response context properties](#response-context).
   * `"session_state"`: _Optional_. An object containing the "memory" for the chat app, such as a user ID.
* `"created"`: The Unix timestamp (in seconds) of when the chat completion was created. _Comes from the OpenAI response._
* `"id"`: A unique identifier for the chat completion. _Comes from the OpenAI response._
* `"model"`: The model used for the completion, such as "gpt-35-turbo". _Comes from the OpenAI response._
* `"object"`: The object type, always "chat.completion". _Comes from the OpenAI response._
* `"prompt_filter_results"`: An object from the Azure Content Safety filter. _Comes from the OpenAI response, but will only be returned when using Azure OpenAI, not openai.com OpenAI._
* `"system_fingerprint"`: Represents the backend configuration that the model runs with. _Comes from the OpenAI response._
* `"usage"`: Usage statistics for the completion request. _Comes from the OpenAI response._


Here's an example of the first three JSON objects in a streaming response:

```json
{
    "choices": [
        {
            "delta": {
                "role": "assistant"
            },
            "context": {
                "data_points": {
                    "text": [
                        "Benefit_Options.pdf#page=3:  The plans also cover preventive care services such as mammograms, colonoscopies, and  other cancer screenings...(truncated)",
                        "Benefit_Options.pdf#page=3:   Both plans offer coverage for medical services. Northwind Health Plus offers coverage for hospital stays,  doctor visits,...(truncated)",
                        "Benefit_Options.pdf#page=3:  With Northwind Health Plus, you can choose  from a variety of in -network providers, including primary care physicians,...(truncated)"
                    ]
                },
                "thoughts": [
                    {
                        "title": "Original user query",
                        "description": "What is included in my Northwind Health Plus plan that is not in standard?",
                        "props": null
                    },
                    {
                        "title": "Generated search query",
                        "description": "Northwind Health Plus plan standard",
                        "props": {
                            "use_semantic_captions": false,
                            "has_vector": false
                        }
                    },
                    {
                        "title": "Results",
                        "description": [
                            {
                                "id": "file-Benefit_Options_pdf-42656E656669745F4F7074696F6E732E706466-page-2",
                                "content": " The plans also cover preventive care services such as mammograms, colonoscopies, and \nother cancer screenings...(truncated)",
                                "embedding": null,
                                "imageEmbedding": null,
                                "category": null,
                                "sourcepage": "Benefit_Options.pdf#page=3",
                                "sourcefile": "Benefit_Options.pdf",
                                "oids": [],
                                "groups": [],
                                "captions": []
                            },
                            {
                                "id": "file-Benefit_Options_pdf-42656E656669745F4F7074696F6E732E706466-page-3",
                                "content": " \nBoth plans offer coverage for medical services. Northwind Health Plus offers coverage for hospital stays, \ndoctor visits,...(truncated)",
                                "embedding": null,
                                "imageEmbedding": null,
                                "category": null,
                                "sourcepage": "Benefit_Options.pdf#page=3",
                                "sourcefile": "Benefit_Options.pdf",
                                "oids": [],
                                "groups": [],
                                "captions": []
                            },
                            {
                                "id": "file-Benefit_Options_pdf-42656E656669745F4F7074696F6E732E706466-page-1",
                                "content": " With Northwind Health Plus, you can choose \nfrom a variety of in -network providers, including primary care physicians,...(truncated)",
                                "embedding": null,
                                "imageEmbedding": null,
                                "category": null,
                                "sourcepage": "Benefit_Options.pdf#page=3",
                                "sourcefile": "Benefit_Options.pdf",
                                "oids": [],
                                "groups": [],
                                "captions": []
                            }
                        ],
                        "props": null
                    },
                    {
                        "title": "Prompt",
                        "description": [
                            "{'role': 'system', 'content': \"Assistant helps the company employees with their healthcare plan questions, and questions about the employee handbook. Be brief in your answers.\\n        Answer ONLY with the facts listed in the list of sources below. If there isn't enough information below, say you don't know. Do not generate answers that don't use the sources below. If asking a clarifying question to the user would help, ask the question.\\n        For tabular information return it as an html table. Do not return markdown format. If the question is not in English, answer in the language used in the question.\\n        Each source has a name followed by colon and the actual information, always include the source name for each fact you use in the response. Use square brackets to reference the source, for example [info1.txt]. Don't combine sources, list each source separately, for example [info1.txt][info2.pdf].\\n        \\n        \\n        \"}",
                            "{'role': 'user', 'content': 'What is included in my Northwind Health Plus plan that is not in standard?'}",
                            "{'role': 'assistant', 'content': 'There is no specific information provided about what is included in the Northwind Health Plus plan that is not in the standard plan. It is recommended to read the plan details carefully and ask questions to understand the specific benefits of the Northwind Health Plus plan [Northwind_Standard_Benefits_Details.pdf#page=91].'}",
                            "{'role': 'user', 'content': \"What is included in my Northwind Health Plus plan that is not in standard?\\n\\nSources:\\nBenefit_Options.pdf#page=3:  The plans also cover preventive care services such as mammograms, colonoscopies, and  other cancer screenings...(truncated)\\nBenefit_Options.pdf#page=3:   Both plans offer coverage for medical services. Northwind Health Plus offers coverage for hospital stays,  doctor visits,...(truncated)\\nBenefit_Options.pdf#page=3:  With Northwind Health Plus, you can choose  from a variety of in -network providers, including primary care physicians,...(truncated)\"}"
                        ],
                        "props": null
                    }
                ]
            },
            "session_state": null,
            "finish_reason": null,
            "index": 0
        }
    ],
    "object": "chat.completion.chunk"
}{
    "id": "chatcmpl-8lNX48CGv9kW7vXTCa9Jb0J4RfnlQ",
    "choices": [
        {
            "delta": {
                "content": null,
                "function_call": null,
                "role": "assistant",
                "tool_calls": null
            },
            "finish_reason": null,
            "index": 0,
            "content_filter_results": {}
        }
    ],
    "created": 1706302566,
    "model": "gpt-35-turbo",
    "object": "chat.completion.chunk",
    "system_fingerprint": null
}{
    "id": "chatcmpl-8lNX48CGv9kW7vXTCa9Jb0J4RfnlQ",
    "choices": [
        {
            "delta": {
                "content": "The",
                "function_call": null,
                "role": null,
                "tool_calls": null
            },
            "finish_reason": null,
            "index": 0,
            "content_filter_results": {
                "hate": {
                    "filtered": false,
                    "severity": "safe"
                },
                "self_harm": {
                    "filtered": false,
                    "severity": "safe"
                },
                "sexual": {
                    "filtered": false,
                    "severity": "safe"
                },
                "violence": {
                    "filtered": false,
                    "severity": "safe"
                }
            }
        }
    ],
    "created": 1706302566,
    "model": "gpt-35-turbo",
    "object": "chat.completion.chunk",
    "system_fingerprint": null
}
```

#### Error response

If an error is encountered before the stream begins, then the response may look like a non-streaming error response. However, if an error is encountered during the stream, then the server will have already sent a 200 response, and will send a chunk with an error object. Typically that would be the last chunk, but it may not be.

Here's an example of an error chunk:

```json
{
    "error": "The app encountered an error processing your request.\nIf you are an administrator of the app, view the full error in the logs."
}
```


### Answer formatting

To support the display of citations, the answer from the LLM should contain source information in square brackets, such as `[info1.txt]`.

Here's a full example of an answer with citation:

```
There is no specific information provided about what is included in the Northwind Health Plus plan that is not in the standard plan. It is recommended to read the plan details carefully and ask questions to understand the specific benefits of the Northwind Health Plus plan [Northwind_Standard_Benefits_Details.pdf#page=91].
```

### Recommended response context

The response context object can contain any properties. However, here are some recommended properties:

* `"followup_questions"`: A list of follow-up questions to ask the user. 

    Example:

    ```json
    "followup_questions": [
        "What types of prescription drugs are covered?",
        "Which services have lower out-of-pocket costs?"
    ]
    ```

    If a client receives this property and the user has requested follow-up questions, the client should prompt the user with clickable versions of the questions. [See image](images/followup.png)

* `"data_points"`: An object containing text and/or image data chunks, a list in the `"text"` or `"images"` properties.

    Example:

    ```json
    "data_points": {
        "text": [
            "Northwind_Standard_Benefits_Details.pdf#page=91:    Tips for Avoiding Intentionally False Or Misleading Statements:   When it comes to understanding a health plan, it is important to be aware of any intentionally false or misleading statements that the plan provider may make...(truncated)",
            "Northwind_Standard_Benefits_Details.pdf#page=91:  It is important to research the providers and services offered in the Northwind Standard plan in order to  determine if the providers and services offered are sufficient for the employee's needs...(truncated)",
            "Northwind_Standard_Benefits_Details.pdf#page=17:  Employees should keep track of their claims and follow up with  Northwind Health if a claim is not processed in a timely manner...(truncated)"
        ]
    },
    ```

    Example with images:

    ```json
    "data_points": {
        "images": [
            {
                "detail": "auto",
                "url": "data:image/png;base64,iVBOR1BORw0KGgoAAAANSUhEUgAAAAEAAAABAQAAAAA3bvkkAAAACklEQVR4nGMAAQAABQABDQ0tuhsAAAAASUVORK5CYII="
            }
        ],
        "text": [
            "Financial Market Analysis Report 2023-6.png: 3</td><td>1</td></tr></table> Financial markets are interconnected, with movements in one segment often influencing other...(truncated)"
        ]
    },
    ```

    If a client receives this property, the client should display the data points in a perusable format. [See image](images/datapoints.png)

* `"thoughts"`: A list describing each step of the backend. Each step should contain:
    * `"title"`: A string describing the step.
    * `"description"`: A string or list of strings describing the step.
    * `"props"`: _Optional_. An object containing additional properties for the step.

    Example:

    ```json
    "thoughts": [
        {
            "title": "Original user query",
            "description": "What is included in my Northwind Health Plus plan that is not in standard?",
            "props": null
        },
        {
            "title": "Generated search query",
            "description": "Northwind Health Plus plan coverage details",
            "props": {
                "has_vector": false,
                "use_semantic_captions": false
            }
        },
        {
            "title": "Results",
            "description": [
                {
                    "captions": [],
                    "category": null,
                    "content": "  \n\u2022 Understand your coverage limits, and know what services are  covered and what services \nare not covered...(truncated)",
                    "embedding": null,
                    "groups": [],
                    "id": "file-Northwind_Health_Plus_Benefits_Details_pdf-4E6F72746877696E645F4865616C74685F506C75735F42656E65666974735F44657461696C732E706466-page-249",
                    "imageEmbedding": null,
                    "oids": [],
                    "sourcefile": "Northwind_Health_Plus_Benefits_Details.pdf",
                    "sourcepage": "Northwind_Health_Plus_Benefits_Details.pdf#page=100"
                },
                {
                    "captions": [],
                    "category": null,
                    "content": " Employees should keep track of their claims and follow up with \nNorthwind Health if a claim is not processed in a timely manner...(truncated)",
                    "embedding": null,
                    "groups": [],
                    "id": "file-Northwind_Standard_Benefits_Details_pdf-4E6F72746877696E645F5374616E646172645F42656E65666974735F44657461696C732E706466-page-41",
                    "imageEmbedding": null,
                    "oids": [],
                    "sourcefile": "Northwind_Standard_Benefits_Details.pdf",
                    "sourcepage": "Northwind_Standard_Benefits_Details.pdf#page=17"
                },
                {
                    "captions": [],
                    "category": null,
                    "content": " It is important to talk to your doctor or \nhealth care provider to make su re that you understand the details of the clinical trial before \nyou decide to participate...(truncated)",
                    "embedding": null,
                    "groups": [],
                    "id": "file-Northwind_Health_Plus_Benefits_Details_pdf-4E6F72746877696E645F4865616C74685F506C75735F42656E65666974735F44657461696C732E706466-page-57",
                    "imageEmbedding": null,
                    "oids": [],
                    "sourcefile": "Northwind_Health_Plus_Benefits_Details.pdf",
                    "sourcepage": "Northwind_Health_Plus_Benefits_Details.pdf#page=24"
                }
            ],
            "props": null
        },
        {
            "title": "Prompt",
            "description": [
                "{'role': 'system', 'content': 'Assistant helps the company employees with their healthcare plan questions, and questions about the employee handbook. Be brief in your answers.\\n        Answer ONLY with the facts listed in the list of sources below. If there isn\\'t enough information below, say you don\\'t know. Do not generate answers that don\\'t use the sources below. If asking a clarifying question to the user would help, ask the question.\\n        For tabular information return it as an html table. Do not return markdown format. If the question is not in English, answer in the language used in the question.\\n        Each source has a name followed by colon and the actual information, always include the source name for each fact you use in the response. Use square brackets to reference the source, for example [info1.txt]. Don\\'t combine sources, list each source separately, for example [info1.txt][info2.pdf].\\n        Generate 3 very brief follow-up questions that the user would likely ask next.\\n    Enclose the follow-up questions in double angle brackets. Example:\\n    <<Are there exclusions for prescriptions?>>\\n    <<Which pharmacies can be ordered from?>>\\n    <<What is the limit for over-the-counter medication?>>\\n    Do no repeat questions that have already been asked.\\n    Make sure the last question ends with \">>\".\\n    \\n        \\n        '}",
                "{'role': 'user', 'content': 'What is included in my Northwind Health Plus plan that is not in standard?'}",
                "{'role': 'assistant', 'content': 'The Northwind Health Plus plan includes coverage for prescription drugs, but it is important to read the plan details to determine which prescription drugs are covered and what the associated costs are [Northwind_Standard_Benefits_Details.pdf#page=91]. Additionally, employees should select in-network providers to maximize coverage and avoid unexpected costs, submit claims as soon as possible after a service is rendered, and track claims and follow up with Northwind Health if a claim is not processed in a timely manner [Northwind_Standard_Benefits_Details.pdf#page=17].\\n\\n'}",
                "{'role': 'user', 'content': 'What is included in my Northwind Health Plus plan that is not in standard?\\n\\nSources:\\nNorthwind_Health_Plus_Benefits_Details.pdf#page=100:    \u2022 Understand your coverage limits, and know what services are  covered and what services  are not covered...(truncated)\\nNorthwind_Standard_Benefits_Details.pdf#page=17:  Employees should keep track of their claims and follow up with  Northwind Health if a claim is not processed in a timely manner...(truncated)\\nNorthwind_Health_Plus_Benefits_Details.pdf#page=24:  It is important to talk to your doctor or  health care provider..(truncated)'}"
            ],
            "props": null
        }
    ]
    ```

    If a client receives this property, the client should display the thoughts in a debug display or to the end-user as specified by the design system. [See image](images/thoughts.png)

## Sample applications

The following applications support at least a subset of the schema described above:

Markdown table:

| Application | Supports schema | Description |
| ----------- | --------------- | ----------- |
| [azure-search-openai-demo](https://www.github.com/Azure-samples/azure-search-openai-demo) | Supports the full schema, including all recommended context properties. | A RAG chat app that uses Azure AI Search and OpenAI. |
| [azure-search-openai-javascript](https://www.github.com/Azure-samples/azure-search-openai-javascript) | Supports most of the schema, but not all recommended context properties (like for GPT-4V feature). | A RAG chat app that uses Azure AI Search and OpenAI. |
| [chatgpt-backend-fastapi](https://github.com/pamelafox/chatgpt-backend-fastapi/) | Supports messages and stream, but no context properties. | A simple chat app using OpenAI. |
| [chatgpt-quickstart](https://github.com/Azure-Samples/chatgpt-quickstart) | Supports messages, but no context properties, and only supports stream = True. | A simple chat app using OpenAI. |
