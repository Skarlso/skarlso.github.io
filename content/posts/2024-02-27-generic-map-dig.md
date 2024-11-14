+++
author = "hannibal"
categories = ["golang", "generics", "typed-parameters"]
date = "2024-02-27T01:01:00+01:00"
title = "Generic dig for map key using typed parameters"
url = "/2024/02/27/generic-map-dig"
comments = true
+++

# Generic dig for map key using typed parameters

Hello!

I was fiddling with a way of getting out values from a map that is of format `map[string]any`.
But I wanted my type safety as well. This was coming from digging out keys from a Metadata field.

The metadata was in a JSON format.

This is what I came up with:

```go
/ FetchValueFromMetadata fetches a key from a metadata if it exists. It will recursively look in
// embedded values as well. Must be a unique key, otherwise it will just return the first
// occurrence.
func FetchValueFromMetadata[T any](key string, data *apiextensionsv1.JSON, def T) (t T, _ error) {
	if data == nil {
		return def, nil
	}

	m := map[string]any{}
	if err := json.Unmarshal(data.Raw, &m); err != nil {
		return t, fmt.Errorf("failed to parse JSON raw data: %w", err)
	}

	v, err := dig[T](key, m)
	if err != nil {
		if errors.Is(err, errKeyNotFound) {
			return def, nil
		}
	}

	return v, nil
}

func dig[T any](key string, data map[string]any) (t T, _ error) {
	if v, ok := data[key]; ok {
		c, k := v.(T)
		if !k {
			return t, fmt.Errorf("failed to convert value to the desired type; was: %T", v)
		}

		return c, nil
	}

	for _, v := range data {
		if ty, ok := v.(map[string]any); ok {
			return dig[T](key, ty)
		}
	}

	return t, errKeyNotFound
}
```

The interesting part is the `dig` method and the type assert to the desired part. Calling this with something like:

```go
value, err := FetchValueFromMetadata[int]("my-key", data, -1)
if err != nil {
  return err
}

fmt.Println(value) // value is of type int
```

will result in the desired value with the right type. The other interesting part is the default value in case the key is
not found. That's just a nice addition, I think.

_Note_: The JSON marshaller in Go will always consider number types as `float64`. But if you just happened to have a map
with some arbitrary data, the `dig` part is still neat.
