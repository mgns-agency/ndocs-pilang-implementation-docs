# Query NDocs API and use PiLang for strings

Implementation is a work in progress and not final.

More information on the API:s
- https://api.nordan.no/ndocs/spec/
- https://api.nordan.no/lang/spec/

## General flow for implementation

Create query -> NDocs -> PiLang -> Result

## Example implementation using PHP

### $queryString example
`documents?filter[language]=se&filter[marketStandard]=SE&filter[market]=SE&filter[productSeries]=SER.MSSE.BOR&filter[category]=document_OPP`

### queryNDocsApi

```php
public function queryNDocsApi(string $queryString)
{
    $apiUrl = NDocs::$plugin->getSettings()->apiUrl;
    $endpoint = $apiUrl . '/ndocs/' . $queryString . '&page[limit]=999';

    $headers = [
        'Accept' => 'application/vnd.api+json'
    ];

    parse_str(parse_url($queryString)['query'], $parsedQuery);
    $language = $parsedQuery['filter']['language'];

    if (strlen($language) !== 2) {
        $language = 'se'; // Fallback language
    }

    try {
        $client = new Client();
        $request = new Request('GET', $endpoint, $headers);
        $response = $client->send($request);
        $result = $response->getBody()->getContents();
        return $this->translateCategory($result, $language);
    } catch (\Exception $e) {
        Craft::error($e->getMessage());
        return false;
    }
}
```

### getPiLangStrings 

```php
private function getPiLangStrings(string $language)
{
    $apiUrl = NDocs::$plugin->getSettings()->apiUrl;
    $groups = 'piProductAttribute_Name,piDocumentCategory_Name';
    $endpoint = $apiUrl . '/lang/strings?lang=' . $language . '&groups=' . $groups;

    $headers = [
        'Accept' => 'application/vnd.api+json'
    ];

    try {
        $client = new Client();
        $request = new Request('GET', $endpoint, $headers);
        $response = $client->send($request);
        $result = $response->getBody()->getContents();
        return $result;
    } catch (\Exception $e) {
        Craft::error($e->getMessage());
        return false;
    }
}
```

### translateCategory

```php
private function translateCategory(string $result, string $language): array
{

    $strings = $this->getPiLangStrings($language);
    if (!$strings) return $result;
    $translatedResult = json_decode($result)->data;

    foreach ($translatedResult as $doc) {
        $cat = $doc->attributes->category;
        $doc->attributes->category = json_decode($strings)->piDocumentCategory_Name->$cat;

        foreach ($doc->attributes->attributes as $index => $attribute) {
            $doc->attributes->attributes[$index] = json_decode($strings)->piProductAttribute_Name->$attribute;
        }
    }

    return $translatedResult;
}
```

