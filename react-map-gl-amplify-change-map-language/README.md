# Changing language of map using MapLibre GL JS with Amazon Location Service and AWS Amplify in a React application

![](https://user-images.githubusercontent.com/7353869/153733699-c5ac8c98-7232-487c-8594-cd13a9090e67.gif)

This example demonstrates how to create a dynamic map using [MapLibre GL JS](https://maplibre.org/maplibre-gl-js-docs/api/)
with [Amazon Location Service](https://aws.amazon.com/location) and [AWS
Amplify](https://aws.amazon.com/amplify/) that allows to change the language of the map within a React application.

[`react-map-gl`](https://visgl.github.io/react-map-gl/) is a set of React components that can be
used to display maps. [`react-map-gl-amplify`](../react-map-gl-amplify/) demonstrates how to use
`react-map-gl` with Amazon Location and AWS Amplify.

## Implementation

To change the language of the map, the application needs to manipulate the style descriptor of the map. Amazon Location Service provides an API called [`GetMapStyleDescriptor`](https://docs.aws.amazon.com/location-maps/latest/APIReference/API_GetMapStyleDescriptor.html) that returns the style descriptor of the map. The style descriptor, once decoded, is a JSON object that contains the map's style and all its layers.

Example:
```json
{
  "version": 8,
  "sources": {
    "esri": {
      "attribution": "Esri, HERE, Garmin, FAO, NOAA, USGS, © OpenStreetMap contributors, and the GIS User Community",
      "maxzoom": 15,
      "tiles": ["https://maps.geo.eu-west-1.amazonaws.com/maps/v0/maps/YourMap-dev/tiles/{z}/{x}/{y}"],
      "type": "vector"
    }
  },
  "sprite": "https://maps.geo.eu-west-1.amazonaws.com/maps/v0/maps/YourMap-dev/sprites/sprites",
  "glyphs": "https://maps.geo.eu-west-1.amazonaws.com/maps/v0/maps/YourMap-dev/glyphs/{fontstack}/{range}.pbf",
  "layers": [{
    "filter": ["==", "_symbol", 0],
    "id": "Land/Not ice",
    "layout": {},
    "minzoom": 0,
    "paint": {
      "fill-color": "#f7f6d5"
    },
    "source": "esri",
    "source-layer": "Land",
    "type": "fill"
  }, {
    "filter": ["==", "_symbol", 1],
    "id": "Land/Ice",
    "layout": {},
    "minzoom": 0,
    "paint": {
      "fill-color": "#FFFFFF"
    },
    "source": "esri",
    "source-layer": "Land",
    "type": "fill"
  }, 
  ...
  ]
}
```

Some of the layers in the style descriptors have the `layout.text-field` property. The value of this property is used to determine which locale is used in the map. Changing the value to a different locale (e.g. `_name_ja` for Japanese) and then passing the new style descriptor to the map will change the language of the map.

```json
{
  "filter": ["==", "_label_class", 0],
  "id": "Water point/Sea or ocean",
  "layout": {
    "icon-allow-overlap": true,
    "icon-image": "Water point",
    "text-anchor": "center",
    "text-field": "{_name_global}",
    "text-font": ["Arial Italic"],
    "text-letter-spacing": 0.1,
    "text-max-width": {
      "stops": [
        [15.5, 7],
        [15.6, 9]
      ]
    },
    "text-padding": 15,
    "text-size": 11
  },
  "minzoom": 9,
  "paint": {
    "text-color": "#497AAB"
  },
  "source": "esri",
  "source-layer": "Water point",
  "type": "symbol"
}
```

Here's a high level implementation example of how to retrieve and manipulate the style descriptor of the map:
```js
import { Geo } from 'aws-amplify';
import { LocationClient, GetMapStyleDescriptorCommand } from '@aws-sdk/client-location';

const textEncoder = new TextDecoder("utf-8");

const getStyleDescriptorJson = async () => {
  const locationClient = new LocationClient({
    region: (Geo.getDefaultMap()).region,
    credentials: await Auth.currentCredentials(), // AWS credentials
  });

  const res = await locationClient.send(new GetMapStyleDescriptorCommand({
    MapName: Geo.getDefaultMap().mapName,
  }));
  if (res.Blob.length === 0) {
    throw new Error("No style descriptor found");
  }
  return JSON.parse(textEncoder.decode(res.Blob));  
};

const language = "_name_ja"; // New language to apply to the map

const styleDescriptorJson = await getStyleDescriptorJson();
styleDescriptorJson.layers = styleDescriptorJson.layers.map((layer) => {
  if (
    layer.layout &&
    layer.layout["text-field"] &&
    ["{_name_global}", "{_name}"].includes(layer.layout["text-field"])
  ) {
    return {
      ...layer,
      layout: {
        ...layer.layout,
        "text-field": [
          "coalesce",
          ["get", language],
          ["get", layer.layout["text-field"].replace(/[{}]/g, "")]
        ]
      }
    };
  }

  return layer;
});

// Then pass the new style descriptor to the map
```

**Note:** Esri maps and Here maps use different values for the `text-field` property. For instance, Japanese is `_name_ja` for Esri maps and `name:ja` for Here maps.

See [`src/App.js`](src/App.js) and [`src/hooks/useLocalizedStyleDescriptor.js`](src/hooks/useLocalizedStyleDescriptor.js) for a full React implementation and a complete list of language codes.

## Dependencies

See [`package.json`](package.json#L6-L13).

## Getting started

1. Install the Amplify CLI: `npm install -g @aws-amplify/cli`
1. Install project dependencies: `npm install`
1. Create a new Amplify project environment: `amplify init`
1. Create Auth and Geo resources: `amplify push`

## Cleaning up

To clean up all resources created for this application, run the following:

```bash
amplify delete
```

## Security

See [CONTRIBUTING](../CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

## Acknowledgements

- [Amazon Location Maps -- Localization of Esri styles](https://observablehq.com/d/d33cd2634b2e2137)
- [Amazon Location Maps -- Localization of HERE styles](https://observablehq.com/d/ae773856b183964e)
