

## Values order of specificity:
--set parameters  >  user-supplied values file  >  parent chart's values.yaml  > values.yaml
```
--set parameters  >  user-supplied values file  >  parent chart's values.yaml  > values.yaml
```

## Deleting a default key  
If you need to delete a key from the default values, 
you may override the value of the key to be null.
```
$ helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```


## Template Functions and Pipelines
Template functions follow the syntax :
```functionName arg1 arg2.... quote ARGUMENT```

### Pipelines
We "sent" the argument to the function using a pipeline (|): 
```.Values.favorite.drink | quote```. Using pipelines, we can chain
```{{ .Values.favorite.drink | quote }}```

Inverting the order is a common practice in templates. You will see ```.val | quote``` more often than quote .val. Either practice is fine.

## default command
the default command is perfect for computed values, which can not be declared inside values.yaml. For example:

```
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

## Using the lookup function
The lookup function can be used to look up resources in a running cluster

When lookup returns an object, it will return a dictionary. This dictionary can be further navigated to extract specific values.

The following example will return the annotations present for the mynamespace object:
(lookup "v1" "Namespace" "" "mynamespace").metadata.annotations

When lookup returns a list of objects, it is possible to access the object list via the items field:
```
{{ range $index, $service := (lookup "v1" "Service" "mynamespace" "").items }}
    {{/* do something with each service */}}
{{ end }}
```


## Template Function List
https://helm.sh/docs/chart_template_guide/function_list/


## Flow Control
- if/else for creating conditional blocks
- with to specify a scope
- range, which provides a "for each"-style loop
- define declares a new named template inside of your template
- template imports a named template
- block declares a special kind of fillable template area
```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

* A pipeline is evaluated as false if the value is:
    * a boolean false
    * a numeric zero
    * an empty string
    * a nil (empty or null)
    * an empty collection (map, slice, tuple, dict, array)



Finally, sometimes it's easier to tell the template system how to indent for you instead of trying to master the spacing of template directives.
```indent```  :
```
function ({{ indent 2 "mug:true" }}).
```


### Modifying scope using with
```
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

use '$' for accessing the object Release.Name from the parent scope.
```
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}
food: {{ .food | upper | quote }}
release: {{ $.Release.Name }}
{{- end }}
```

After looking at range, we will take a look at template variables, which offer one solution to the scoping issue above.

## Looping with the range action


The |- marker in YAML takes a multi-line string. 
This can be a useful technique for embedding big blocks of data inside of your manifests, as exemplified here:

```
toppings: |-
{{- range $.Values.pizzaToppings }}
- {{ . | title | quote }}
{{- end }}    
{{- end }}
```


```sizes: |-
    {{- range tuple "small" "medium" "large" }}
    - {{ . }}
    {{- end }} 
```

The above will produce this:
```
  sizes: |-
    - small
    - medium
    - large     
```

## variables

Note that range comes first, then the variables, then the assignment operator, then the list.
```toppings: |-
{{- range $index, $topping := .Values.pizzaToppings }}
    {{ $index }}: {{ $topping }}
{{- end }}
```

there is one variable that is always global - $ - this variable will always point to the root context. 
```
{{- range .Values.tlsSecrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .name }}
  labels:
    # Many helm templates would use `.` below, but that will not work,
    # however `$` will work here
    app.kubernetes.io/name: {{ template "fullname" $ }}
    # I cannot reference .Chart.Name, but I can do $.Chart.Name
    helm.sh/chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
    app.kubernetes.io/instance: "{{ $.Release.Name }}"
    # Value from appVersion in Chart.yaml
    app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
    app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
type: kubernetes.io/tls
data:
  tls.crt: {{ .certificate }}
  tls.key: {{ .key }}

{{- end }}

```


## Named Templates



One popular naming convention is to prefix each defined template with the name of the chart: ``` {{ define "mychart.labels" }} ```. By using the specific chart name as a prefix we can avoid any conflicts that may arise due to two different charts that implement templates of the same name.

### Partials and _ files

Before we get to the nuts-and-bolts of writing those templates, there is file naming convention that deserves mention:

Most files in templates/ are treated as if they contain Kubernetes manifests
The ``` NOTES.txt ``` is one exception
But files whose name begins with an underscore (_) are assumed to not have a manifest inside. These files are not rendered to Kubernetes object definitions, but are available everywhere within other chart templates for use.


### Declaring and using templates with define and template

The define action allows us to create a named template inside of a template file:

```
{{ define "MY.NAME" }}
  # body of template here
{{ end }}
```

For example, we can define a template to encapsulate a Kubernetes block of labels:
```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```
Now we can embed this template inside of our existing ConfigMap, and then include it with the ```template``` action:

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

## Named and Templates

documentation : ```{{/* ... */}}```
_helpers.tpl
```
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

Even though this definition is in _helpers.tpl, it can still be accessed in configmap.yaml:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

### Setting the scope of a template
 ```{{- template "mychart.labels" }}```
 No scope was passed in, so within the template we cannot access anything in .. This is easy 
 enough to fix, though. We simply pass a scope to the template:
 ```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
 ```


## **Importing Child Values via dependencies**
Using the ```exports``` format

If a child chart's values.yaml file contains an exports field at the root, 
its contents may be imported directly into the parent's values by specifying the keys to import as in the example below:
```
# parent's Chart.yaml file

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    import-values:
      - data
```
```
# child's values.yaml file

exports:
  data:
    myint: 99
```
Since we are specifying the key data in our import list, Helm looks in the exports field of the child chart for data key and imports its contents.

The final parent values would contain our exported field:
```
# parent's values

myint: 99
```

## **Using the child-parent format**
https://helm.sh/docs/topics/charts/#using-the-child-parent-format


