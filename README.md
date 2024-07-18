# LLM- 
I have scraped data from OpenWebText Corpus: https://skylion007.github.io/OpenWebTextCorpus/

The model here has a single head. But, employing 4 layers might be better (4 decoders).

# Research Papers:
https://arxiv.org/abs/2303.18223 , 
https://arxiv.org/abs/1706.03762
import yaml
from jinja2 import Template

def generate_assertions(schema):
    assertions = []
    if isinstance(schema, dict):
        for key, value in schema.get('properties', {}).items():
            if value.get('type') == 'string':
                assertions.append(f'assertNotNull(responseBody.get{key.capitalize()}());')
            elif value.get('type') == 'integer' or value.get('type') == 'number':
                assertions.append(f'assertNotNull(responseBody.get{key.capitalize()}());')
            elif value.get('type') == 'boolean':
                assertions.append(f'assertNotNull(responseBody.is{key.capitalize()}());')
            elif value.get('type') == 'array':
                assertions.append(f'assertNotNull(responseBody.get{key.capitalize()}());')
                assertions.append(f'assertFalse(responseBody.get{key.capitalize()}().isEmpty());')
    return assertions

def generate_pact_test(swagger_file, output_file):
    with open(swagger_file, 'r') as file:
        swagger_data = yaml.safe_load(file)

    paths_methods = []
    for path, path_data in swagger_data['paths'].items():
        for method, operation in path_data.items():
            parameters = operation.get('parameters', [])
            headers = [param['name'] for param in parameters if param['in'] == 'header']
            path_params = [param['name'] for param in parameters if param['in'] == 'path']
            query_params = [param['name'] for param in parameters if param['in'] == 'query']
            request_body_ref = operation.get('requestBody', {}).get('content', {}).get('application/json', {}).get('schema', {}).get('$ref')
            request_body = request_body_ref.split('/')[-1] if request_body_ref else None
            
            responses = {}
            for status_code, response_data in operation.get('responses', {}).items():
                response_schema = response_data.get('content', {}).get('application/json', {}).get('schema', {})
                response_type = response_schema.get('$ref', '').split('/')[-1] or 'Void'
                responses[status_code] = {
                    'type': response_type,
                    'description': response_data.get('description', ''),
                    'schema': response_schema
                }
            
            paths_methods.append({
                'path': path,
                'method': method.upper(),
                'headers': headers,
                'path_params': path_params,
                'query_params': query_params,
                'request_body': request_body,
                'operation_id': operation.get('operationId', f"{method}_{path.replace('/', '_')}"),
                'responses': responses
            })

    schemas = swagger_data.get('components', {}).get('schemas', {})

    imports = set([
        "au.com.dius.pact.consumer.MockServer",
        "au.com.dius.pact.consumer.dsl.PactDslWithProvider",
        "au.com.dius.pact.consumer.junit5.PactConsumerTestExt",
        "au.com.dius.pact.consumer.junit5.PactTestFor",
        "au.com.dius.pact.core.model.RequestResponsePact",
        "au.com.dius.pact.core.model.annotations.Pact",
        "org.junit.jupiter.api.Test",
        "org.junit.jupiter.api.extension.ExtendWith",
        "org.springframework.http.*",
        "org.springframework.web.client.RestTemplate",
        "org.springframework.web.client.HttpClientErrorException",
        "org.springframework.web.client.HttpServerErrorException",
        "java.io.IOException",
        "java.util.HashMap",
        "java.util.Map",
        "static org.junit.jupiter.api.Assertions.*",
        "com.fasterxml.jackson.databind.ObjectMapper",
        "au.com.dius.pact.core.model.PactSpecVersion",
        "au.com.dius.pact.consumer.junit5.ProviderType"
    ])

    for pm in paths_methods:
        if pm['request_body']:
            imports.add(f"com.example.model.{pm['request_body']}")
        for response in pm['responses'].values():
            if response['type'] and response['type'] != 'Void':
                imports.add(f"com.example.model.{response['type']}")

    template = Template("""
package com.example.pact;

{% for import in imports %}
import {{ import }};
{% endfor %}

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "{{ '{{' }}PROVIDER_NAME{{ '}}' }}", pactVersion = PactSpecVersion.V3)
public class ApiPactTest {

    private static final String CONSUMER_NAME = "{{ '{{' }}CONSUMER_NAME{{ '}}' }}";
    private static final ObjectMapper objectMapper = new ObjectMapper();

    {% for pm in paths_methods %}
    {% for status_code, response in pm.responses.items() %}
    @Pact(provider = "{{ '{{' }}PROVIDER_NAME{{ '}}' }}", consumer = CONSUMER_NAME)
    public RequestResponsePact pactFor{{ pm.operation_id | capitalize }}{{ status_code }}(PactDslWithProvider builder) {
        Map<String, String> headers = new HashMap<>();
        {% for header in pm.headers %}
        headers.put("{{ header }}", "{{ header }}_value");
        {% endfor %}

        return builder
            .given("{{ pm.operation_id }} state for {{ status_code }} response")
            .uponReceiving("A request to {{ pm.operation_id }} expecting {{ status_code }}")
            .path("{{ pm.path }}")
            .method("{{ pm.method }}")
            .headers(headers)
            {% if pm.query_params %}
            .query("{{ pm.query_params | join('=value&') }}=value")
            {% endif %}
            {% if pm.request_body %}
            .body(PactDslJsonBody.newJsonBody()
                {% for prop, schema in schemas[pm.request_body]['properties'].items() %}
                    {% if schema['type'] == 'string' %}
                    .stringType("{{ prop }}")
                    {% elif schema['type'] in ['integer', 'number'] %}
                    .numberType("{{ prop }}")
                    {% elif schema['type'] == 'boolean' %}
                    .booleanType("{{ prop }}")
                    {% elif schema['type'] == 'array' %}
                    .array("{{ prop }}")
                    {% endif %}
                {% endfor %}
                .build())
            {% endif %}
            .willRespondWith()
            .status({{ status_code }})
            .body(PactDslJsonBody.newJsonBody()
                {% for prop, schema in schemas[response.type]['properties'].items() %}
                    {% if schema['type'] == 'string' %}
                    .stringType("{{ prop }}")
                    {% elif schema['type'] in ['integer', 'number'] %}
                    .numberType("{{ prop }}")
                    {% elif schema['type'] == 'boolean' %}
                    .booleanType("{{ prop }}")
                    {% elif schema['type'] == 'array' %}
                    .array("{{ prop }}")
                    {% endif %}
                {% endfor %}
                .build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "pactFor{{ pm.operation_id | capitalize }}{{ status_code }}", providerType = ProviderType.SYNCH)
    void test{{ pm.operation_id | capitalize }}{{ status_code }}(MockServer mockServer) throws IOException {
        RestTemplate restTemplate = new RestTemplate();
        String url = mockServer.getUrl() + "{{ pm.path }}";
        {% if pm.path_params %}
        // Replace path parameters
        {% for param in pm.path_params %}
        url = url.replace("{{ '{' }}{{ param }}{{ '}' }}", "{{ param }}_value");
        {% endfor %}
        {% endif %}

        {% if pm.query_params %}
        url += "?{{ pm.query_params | join('=value&') }}=value";
        {% endif %}

        HttpHeaders headers = new HttpHeaders();
        {% for header in pm.headers %}
        headers.set("{{ header }}", "{{ header }}_value");
        {% endfor %}

        {% if pm.request_body %}
        {{ pm.request_body }} requestBody = new {{ pm.request_body }}();
        // Set requestBody properties here
        HttpEntity<{{ pm.request_body }}> requestEntity = new HttpEntity<>(requestBody, headers);
        {% else %}
        HttpEntity<?> requestEntity = new HttpEntity<>(headers);
        {% endif %}

        try {
            ResponseEntity<String> response = restTemplate.exchange(url, HttpMethod.{{ pm.method }}, requestEntity, String.class);
            assertEquals({{ status_code }}, response.getStatusCodeValue());
            {% if response.type != 'Void' %}
            assertNotNull(response.getBody());
            {{ response.type }} responseBody = objectMapper.readValue(response.getBody(), {{ response.type }}.class);
            {% for assertion in generate_assertions(schemas[response.type]) %}
            {{ assertion }}
            {% endfor %}
            {% endif %}
        } catch (HttpClientErrorException | HttpServerErrorException e) {
            assertEquals({{ status_code }}, e.getRawStatusCode());
            {% if response.type != 'Void' %}
            {{ response.type }} errorBody = objectMapper.readValue(e.getResponseBodyAsString(), {{ response.type }}.class);
            {% for assertion in generate_assertions(schemas[response.type]) %}
            {{ assertion }}
            {% endfor %}
            {% endif %}
        }
    }

    {% endfor %}
    {% endfor %}
}
    """)

    java_code = template.render(imports=sorted(imports), paths_methods=paths_methods, schemas=schemas, generate_assertions=generate_assertions)

    with open(output_file, 'w') as file:
        file.write(java_code)

# Usage
generate_pact_test('swagger.yaml', 'ApiPactTest.java')
