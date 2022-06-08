# ENV binder
[![License](http://img.shields.io/:license-apache-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)
[![Go Reference](https://pkg.go.dev/badge/github.com/AbsaOSS/env-binder.svg)](https://pkg.go.dev/github.com/AbsaOSS/env-binder?branch=master)
![Build Status](https://github.com/AbsaOSS/env-binder/actions/workflows/build.yaml/badge.svg?branch=master)
![Linter](https://github.com/AbsaOSS/env-binder/actions/workflows/lint.yaml/badge.svg?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/AbsaOSS/env-binder)](https://goreportcard.com/report/github.com/AbsaOSS/env-binder?branch=master)

The ENV-BINDER package is used to easily bind environment variables to GO structures. ENV-BINDER is designed to
be usable in the widest possible range of scenarios.Among other things, it supports variable
prefixes and bindings to unexported arrays. Take a look at the following usage example:
```golang
import "github.com/AbsaOSS/env-binder/env"

type Endpoint struct {
	URL string `env:"ENDPOINT_URL, require=true"`
}

type Config struct {

	// reading string value from NAME
	Name string `env:"NAME"`

	// reuse an already bound env variable NAME
	Description string `env:"NAME"`

	// reuse an already bound variable NAME, but replace only when name 
	// was not set before
	AlternativeName string `env:"NAME, protected=true"`

	// reading int with 8080 as default value
	DefaultPort uint16 `env:"PORT, default=8080"`

	// reading slice of strings with default values
	Regions []string `env:"REGIONS, default=[us-east-1,us-east-2,us-west-1]"`

	// reading slice of strings from env var
	Subnets []string `env:"SUBNETS, default=[10.0.0.0/24,192.168.1.0/24]"`
	
	// default=[] ensures that if INTERVALS does not exist, 
	// []int8{} is set instead of []int8{nil}
	Interval []uint8 `env:"INTERVALS, default=[]"`

	// nested structure
	Credentials struct {

		// binding required value
		KeyID string `env:"ACCESS_KEY_ID, require=true"`

		// binding to private field
		secretKey string `env:"SECRET_ACCESS_KEY, require=true"`
	}

	// expected PRIMARY_ prefix in nested environment variables
	Endpoint1 Endpoint `env:"PRIMARY"`

	// expected FAILOVER_ prefix in nested environment variables
	Endpoint2 Endpoint `env:"FAILOVER"`

	// the field does not have a bind tag set, 
	// so it will be ignored during bind
	Args []string
}


func main() {
	defer clean()
	os.Setenv("PRIMARY_ENDPOINT_URL", "https://ep1.cloud.example.com")
	os.Setenv("FAILOVER_ENDPOINT_URL", "https://ep2.cloud.example.com")
	os.Setenv("ACCESS_KEY_ID", "AKIAIOSFODNN7EXAMPLE")
	os.Setenv("SECRET_ACCESS_KEY", "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY")
	os.Setenv("NAME", "Hello from 12-factor")
	os.Setenv("PORT", "9000")
	os.Setenv("SUBNETS", "10.0.0.0/24,10.0.1.0/24, 10.1.0.0/24,  10.1.1.0/24")

	c := &Config{}
	c.AlternativeName = "protected name"
	c.Description = "hello from ENV-BINDER"
	if err := env.Bind(c); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(JSONize(c))
}

```
function main generates the following output:
```json
{
  "Name": "Hello from 12-factor",
  "Description": "Hello from 12-factor",
  "AlternativeName": "protected name",
  "DefaultPort": 9000,
  "Regions": [
    "us-east-1",
    "us-east-2",
    "us-west-1"
  ],
  "Subnets": [
    "10.0.0.0/24",
    "10.0.1.0/24",
    "10.1.0.0/24",
    "10.1.1.0/24"
  ],
  "Interval": [],
  "Credentials": {
    "KeyID": "AKIAIOSFODNN7EXAMPLE"
  },
  "Endpoint1": {
    "URL": "https://ep1.cloud.example.com"
  },
  "Endpoint2": {
    "URL": "https://ep2.cloud.example.com"
  },
  "Args": null
}
```

## supported types
ENV-BINDER supports all types listed in the following table.  In addition, it should be noted that in the case
of slices, ENV-BINDER creates an instance of an empty slice if the value of the environment variable is
declared and its value is empty string. In this case ENV-BINDER returns an empty slice instead of the vulnerable nil.

| primitive types | slices |
|---|---|
| `int`,`int8`,`int16`,`int32`,`int64` | `[]int`,`[]int8`,`[]int16`,`[]int32`,`[]int64` |
| `float32`,`float64` | `[]float32`,`[]float64` |
| `uint`,`uint8`,`uint16`,`uint32`,`uint64` | `[]uint`,`[]uint8`,`[]uint16`,`[]uint32`,`[]uint64` |
| `bool` | `[]bool` |
| `string` | `[]string` |

## supported keywords
Besides the fact that ENV-BINDER works with private fields and can add prefixes to variable names, it
operates with several keywords. The structure in the introductory section works with all types
of these keywords.

- `default` - the value specified in the default tag is used in case env variable does not exist. e.g:
  `env: "SUBNET", default=10.0.1.0/24` or `env: "ENV_SUBNETS", default=[]` which will set an empty slice instead
  of a vulnerable nil value in case `ENV_SUBNETS` does not exist.

- `require` - if `require=true` then env variable must exist otherwise Bind function returns error

- `protected` - if `protected=true` then, in case the field in the structure already has a set value , the
  Bind function will not set it. Otherwise, bind will be applied to it.

You can combine individual tags freely: `env: "ENV_SWITCHER", default=[true, false, true], protected=true`
is a perfectly valid configuration

## API
If the Bind function is not enough for you, you can use any of the static functions of our API:
```go
import "github.com/AbsaOSS/env-binder/env"

// GetEnvAsStringOrFallback returns the env variable for the given key
// and falls back to the given defaultValue if not set
GetEnvAsStringOrFallback(key, defaultValue string) string

// GetEnvAsArrayOfStringsOrFallback returns the env variable for the given key
// and falls back to the given defaultValue if not set
// GetEnvAsArrayOfStringsOrFallback trims all whitespaces from input 
// i.e. "us, fr, au" -> {"us","fr","au"}
GetEnvAsArrayOfStringsOrFallback(key string, defaultValue []string) []string

// GetEnvAsIntOrFallback returns the env variable (parsed as integer) for
// the given key and falls back to the given defaultValue if not set
GetEnvAsIntOrFallback(key string, defaultValue int) (int, error)

// GetEnvAsArrayOfIntsOrFallback returns the env variable for the given key
// and falls back to the given defaultValue if not set
GetEnvAsArrayOfIntsOrFallback(key string, defaultValue []int) ([]int, error)


// GetEnvAsFloat64OrFallback returns the env variable (parsed as float64) for
// the given key and falls back to the given defaultValue if not set
GetEnvAsFloat64OrFallback(key string, defaultValue float64) (float64, error)

// GetEnvAsArrayOfFloat64OrFallback returns the env variable for the given key
// and falls back to the given defaultValue if not set
GetEnvAsArrayOfFloat64OrFallback(key string, defaultValue []float64) ([]float64, error)


// GetEnvAsBoolOrFallback returns the env variable for the given key,
// parses it as boolean and falls back to the given defaultValue if not set
GetEnvAsBoolOrFallback(key string, defaultValue bool) (bool, error)

// GetEnvAsArrayOfBoolOrFallback returns the env variable for the given key
// and falls back to the given defaultValue if not set
GetEnvAsArrayOfBoolOrFallback(key string, defaultValue []bool) ([]bool, error) 
```
