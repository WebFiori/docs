# Image Generation

WebFiori AI generates images from text prompts through `generateImage()`. It is supported by OpenAI (DALL·E) and Google (Imagen); see [Providers](learn/ai-providers).

<meta name="description" content="Generate images from text prompts with WebFiori AI using ImageRequest and generateImage on OpenAI and Google.">

## Generating an Image

Build an `ImageRequest` and pass it to `generateImage()`, which returns an `ImageResponse`:

```php
use WebFiori\Ai\ImageRequest;
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;

$client = new OpenAIClient(new OpenAIClientConfig(apiKey: 'sk-...'));

$request = new ImageRequest(
    prompt: 'A serene Japanese garden with a bridge over a koi pond, watercolor style',
    size: '1024x1024',
    quality: 'hd',
    style: 'natural',
);

$response = $client->generateImage($request);

$image = $response->getImages()[0];
echo $image->getUrl().PHP_EOL;

if ($image->getRevisedPrompt() !== null) {
    echo 'Revised prompt: '.$image->getRevisedPrompt().PHP_EOL;
}
```

## The Request

`ImageRequest` accepts the prompt plus optional generation parameters:

| Parameter        | Description                                              |
|------------------|----------------------------------------------------------|
| `prompt`         | The text description of the image (required).            |
| `size`           | Output dimensions, e.g. `1024x1024`.                     |
| `quality`        | Quality hint, e.g. `standard` or `hd`.                   |
| `style`          | Style hint, e.g. `natural` or `vivid`.                   |
| `count`          | Number of images to generate.                            |
| `negativePrompt` | What to avoid (provider-dependent).                      |

Provider support for individual parameters varies; the prompt is always required.

## Reading the Response

`ImageResponse::getImages()` returns an array of `GeneratedImage` objects. Each one exposes `getUrl()` for the image URL when the provider returns one, `getBase64()` for base64-encoded data when the image is returned inline, and `getRevisedPrompt()` for the provider's rewritten prompt when available.

```php
foreach ($response->getImages() as $image) {
    if ($image->getUrl() !== null) {
        // download or display via URL
    } elseif ($image->getBase64() !== null) {
        file_put_contents('out.png', base64_decode($image->getBase64()));
    }
}
```

## Unsupported Providers

Anthropic and AWS Bedrock do not provide image generation; calling `generateImage()` on them throws `UnsupportedFeatureException`. Use OpenAI or Google instead.

## Where to Next

See [Embeddings](learn/ai-embeddings) to turn text into vectors for search, or [Providers](learn/ai-providers) for the feature matrix across providers.
