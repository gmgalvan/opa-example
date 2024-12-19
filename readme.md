# Open Policy Agent (OPA) con Gatekeeper en Kubernetes ejemplo

Este documento describe cómo instalar y configurar Open Policy Agent (OPA) con Gatekeeper en un clúster de Kubernetes, y cómo implementar una política que bloquee Pods que no incluyan las etiquetas obligatorias `team` y `environment`.

---

## Prerrequisitos

1. Un clúster de Kubernetes funcional. (Minikiube o Kind)
2. Acceso a `kubectl` con permisos administrativos.
3. (Opcional) Istio configurado correctamente si está presente en el clúster.

---

## Instalación de OPA Gatekeeper

1. Instalar Gatekeeper:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.11.0/deploy/gatekeeper.yaml
   ```

2. Verificar que los pods estén en ejecución:

   ```bash
   kubectl get pods -n gatekeeper-system
   ```

   Todos los pods en el namespace `gatekeeper-system` deberían estar en estado `Running`.

---

## Implementación de una Política: Etiquetas Obligatorias

### 1. Crear el ConstraintTemplate

Crea un archivo llamado `mandatory-labels-template.yaml` con el siguiente contenido:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: mandatorylabels
spec:
  crd:
    spec:
      names:
        kind: MandatoryLabels
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package mandatory_labels

        violation[{"msg": msg}] {
          input.review.object.kind == "Pod"
          missing_label := required_labels[_]
          not input.review.object.metadata.labels[missing_label]
          msg := sprintf("El Pod no tiene la etiqueta requerida: %s", [missing_label])
        }

        required_labels := {"team", "environment"}
```

Aplica el ConstraintTemplate:

```bash
kubectl apply -f mandatory-labels-template.yaml
```

---

### 2. Crear el Constraint

Crea un archivo llamado `mandatory-labels-constraint.yaml` con el siguiente contenido:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: MandatoryLabels
metadata:
  name: enforce-labels
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

Aplica el Constraint:

```bash
kubectl apply -f mandatory-labels-constraint.yaml
```

El archivo ConstraintTemplate es donde defines cómo se validarán las políticas usando el lenguaje Rego. Piensa en este archivo como una "plantilla" que establece:

- La lógica base que evaluará las solicitudes de Kubernetes.
- Las reglas y condiciones en el lenguaje Rego.
- Los datos requeridos para evaluar las políticas.

### Ejemplo:
En nuestro caso, el archivo mandatory-labels-template.yaml contiene la lógica en Rego que valida si un Pod tiene etiquetas específicas (team y environment). Este archivo define un "esqueleto" que puede ser reutilizado para aplicar la misma validación en distintos recursos o con diferentes parámetros.

---

### 3. Probar la Política

#### 3.1 Caso Bloqueado: Pod Sin Etiquetas

Crea un archivo llamado `pod-missing-labels.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: missing-labels
spec:
  containers:
    - name: nginx
      image: nginx:1.21
```

El archivo Constraint utiliza el ConstraintTemplate para aplicar una política a recursos concretos en el clúster. Piensa en este archivo como la "instanciación" de la plantilla que:

- Especifica a qué tipos de recursos Kubernetes (como Pods o Namespaces) se aplicará la lógica.
- Define reglas adicionales, como los match para seleccionar objetos específicos.

### Ejemplo:
En el archivo mandatory-labels-constraint.yaml, usamos el MandatoryLabels (definido en el ConstraintTemplate) para aplicar esta política únicamente a los Pods en el clúster


Aplica el Pod:

```bash
kubectl apply -f pod-missing-labels.yaml
```

Deberías ver un error similar a:

```
Error from server: admission webhook "validation.gatekeeper.sh" denied the request: El Pod no tiene la etiqueta requerida: team
```

#### 3.2 Caso Permitido: Pod Con Etiquetas

Crea un archivo llamado `pod-with-labels.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-labels
  labels:
    team: "devops"
    environment: "staging"
spec:
  containers:
    - name: nginx
      image: nginx:1.21
```

Aplica el Pod:

```bash
kubectl apply -f pod-with-labels.yaml
```

Deberías ver el siguiente resultado:

```
pod/with-labels created
```

---

### 4. Auditoría de Recursos Existentes (Opcional)

Para auditar recursos existentes y verificar si cumplen con las políticas configuradas:

```bash
kubectl get constraints
kubectl get mandatorylabels.constraints.gatekeeper.sh/enforce-labels -o yaml
```

Esto mostrará cualquier recurso que viole las reglas definidas.

---

## Resolución de Problemas

### Istio Webhook No Disponible

Si Istio está causando problemas, puedes deshabilitar la inyección automática de sidecars en el namespace por defecto:

```bash
kubectl label namespace default istio-injection-
```

---

## Conclusión

Con esta configuración, OPA Gatekeeper ayuda a garantizar que todos los Pods incluyan etiquetas obligatorias, mejorando la trazabilidad y el cumplimiento de estándares en el clúster. Este ejemplo puede adaptarse fácilmente para validar otros tipos de configuraciones.

Para más información, consulta la [documentación oficial de Gatekeeper](https://open-policy-agent.github.io/gatekeeper/).


## Referencias
https://stackoverflow.com/questions/67391066/why-is-the-exact-difference-between-violation-and-deny-in-opa-rego
https://www.openpolicyagent.org/docs/latest/
https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/
https://www.openpolicyagent.org/docs/latest/#rego
https://open-policy-agent.github.io/gatekeeper/website/
https://www.openpolicyagent.org/docs/latest/envoy-introduction/