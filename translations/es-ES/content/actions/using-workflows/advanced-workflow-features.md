---
title: Características avanzadas de los flujos de trabajo
shortTitle: Características avanzadas de los flujos de trabajo
intro: 'Esta guía te muestra cómo utilizar las características avanzadas de las {% data variables.product.prodname_actions %}, con administración de secretos, jobs dependientes, almacenamiento en caché, matrices de compilación, ambientes y etiquetas.'
redirect_from:
  - /actions/learn-github-actions/managing-complex-workflows
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
type: how_to
topics:
  - Workflows
miniTocMaxHeadingLevel: 4
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

## Resumen

Este artículo describe algunas de las características avanzadas de las {% data variables.product.prodname_actions %} que te ayudan a crear flujos de trabajo más complejos.

## Almacenar secretos

Si tus flujos de trabajo utilizan datos sensibles tales como contraseñas o certificados, puedes guardarlos en {% data variables.product.prodname_dotcom %} como _secretos_ y luego usarlos en tus flujos de trabajo como variables de ambiente. Esto significa que podrás crear y compartir flujos de trabajo sin tener que embeber valores sensibles directamente en el flujo de trabajo de YAML.

Esta acción de ejemplo ilustra cómo referenciar un secreto existente como una variable de ambiente y enviarla como parámetro a un comando de ejemplo.

{% raw %}
```yaml
jobs:
  example-job:
    runs-on: ubuntu-latest
    steps:
      - name: Retrieve secret
        env:
          super_secret: ${{ secrets.SUPERSECRET }}
        run: |
          example-command "$super_secret"
```
{% endraw %}

Para obtener más información, consulta "[Crear y almacenar secretos cifrados](/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)".

## Crear jobs dependientes

Predeterminadamente, los jobs en tu flujo de trabajo se ejecutan todos en paralelo y al mismo tiempo. Así que, si tienes un job que solo debe ejecutarse después de que se complete otro job, puedes utilizar la palabra clave `needs` para crear esta dependencia. Si un de los jobs falla, todos los jobs dependientes se omitirán; sin embargo, si necesitas que los jobs sigan, puedes definir esto utilizando la declaración condicional [`if`](/actions/using-jobs/using-conditions-to-control-job-execution).

En este ejemplo, los jobs de `setup`, `build`, y `test` se ejecutan en serie, y `build` y `test` son dependientes de que el job que las precede se complete con éxito:

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - run: ./setup_server.sh
  build:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - run: ./build_server.sh
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ./test_server.sh
```

Para obtener más información, consulta la sección "[Definir los jobs de prerrequisito](/actions/using-jobs/using-jobs-in-a-workflow#defining-prerequisite-jobs)".

## Utilizar una matriz de compilaciones

Puedes utilizar una matriz de compilaciones si quieres que tu flujo de trabajo ejecute pruebas a través de varias combinaciones de sistemas operativos, plataformas y lenguajes. La matriz de compilaciones se crea utilizando la palabra clave `strategy`, la cual recibe las opciones de compilación como un arreglo. Por ejemplo, esta matriz de compilaciones ejecutará el job varias veces, utilizando diferentes versiones de Node.js:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [6, 8, 10]
    steps:
      - uses: {% data reusables.actions.action-setup-node %}
        with:
          node-version: {% raw %}${{ matrix.node }}{% endraw %}
```

Para obtener más información, consulte la sección "[Utilizar una matriz de compilación para tus jobs](/actions/using-jobs/using-a-build-matrix-for-your-jobs)".

{% ifversion fpt or ghec %}
## Almacenar dependencias en caché

Los ejecutores hospedados en {% data variables.product.prodname_dotcom %} se inician como ambientes nuevos para cada job, así que, si tus jobs utilizan dependencias a menudo, puedes considerar almacenar estos archivos en caché para ayudarles a mejorar el rendimiento. Una vez que se crea el caché, estará disponible para todos los flujos de trabajo en el mismo repositorio.

Este ejemplo ilustra cómo almacenar el directorio `~/.npm` en el caché:

```yaml
jobs:
  example-job:
    steps:
      - name: Cache node modules
        uses: {% data reusables.actions.action-cache %}
        env:
          cache-name: cache-node-modules
        with:
          path: ~/.npm
          key: {% raw %}${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}{% endraw %}
          restore-keys: |
            {% raw %}${{ runner.os }}-build-${{ env.cache-name }}-{% endraw %}
```

Para obtener más información, consulta la sección "<a href="/actions/guides/caching-dependencies-to-speed-up-workflows" class="dotcom-only">Almacenar las dependencias en caché para agilizar los flujos de trabajo</a>".
{% endif %}

## Usar bases de datos y contenedores de servicio

Si tu job requiere de un servicio de caché o de base de datos, puedes utilizar la palabra clave [`services`](/actions/using-jobs/running-jobs-in-a-container) para crear un contenedor efímero para almacenar el servicio; el contenedor resultante estará entonces disponible para todos los pasos de ese job y se eliminará cuando el job se haya completado. Este ejemplo ilustra como un job puede utilizar `services` para crear un contenedor de `postgres` y luego utilizar a `node` para conectarse al servicio.

```yaml
jobs:
  container-job:
    runs-on: ubuntu-latest
    container: node:10.18-jessie
    services:
      postgres:
        image: postgres
    steps:
      - name: Check out repository code
        uses: {% data reusables.actions.action-checkout %}
      - name: Install dependencies
        run: npm ci
      - name: Connect to PostgreSQL
        run: node client.js
        env:
          POSTGRES_HOST: postgres
          POSTGRES_PORT: 5432
```

Para obtener más información, consulta la sección "[Utilizar bases de datos y contenedores de servicios](/actions/configuring-and-managing-workflows/using-databases-and-service-containers)".

## Utilizar etiquetas para enrutar los flujos de trabajo

Esta característica te ayuda a asignar jobs a un ejecutor hospedado específico. Si quieres asegurarte de que un tipo específico de ejecutor procesará tu job, puedes utilizar etiquetas para controlar donde se ejecutan los jobs. Puedes asignar etiquetas a un ejecutor auto-hospedado adicionalmente a su etiqueta predeterminada de `self-hosted`. Entonces, puedes referirte a estas etiquetas en tu flujo de trabajo de YAML, garantizando que el job se enrute de forma predecible.{% ifversion not ghae %}Los ejecutores hospedados en {% data variables.product.prodname_dotcom %} tienen asignadas etiquetas predefinidas.{% endif %}

Este ejemplo muestra como un flujo de trabajo puede utilizar etiquetas para especificar el ejecutor requerido:

```yaml
jobs:
  example-job:
    runs-on: [self-hosted, linux, x64, gpu]
```

Un flujo de trabajo solo se ejecutará en un ejecutor que tenga todas las etiquetas en el arreglo `runs-on`. El job irá preferencialmente a un ejecutor auto-hospedado inactivo con las etiquetas especificadas. Si no hay alguno disponible y existe un ejecutor hospedado en {% data variables.product.prodname_dotcom %} con las etiquetas especificadas, el job irá a un ejecutor hospedado en {% data variables.product.prodname_dotcom %}.

Para aprender más acerca de las etiquetas auto-hospedadas, consulta la sección ["Utilizar etiquetas con los ejecutores auto-hospedados](/actions/hosting-your-own-runners/using-labels-with-self-hosted-runners)".

{% ifversion fpt or ghes %}
Para aprender más sobre las etiquetas de los ejecutores hospedados en {% data variables.product.prodname_dotcom %}, consulta la sección ["Ejecutores y recursos de hardware compatibles"](/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources).
{% endif %}

{% ifversion fpt or ghes > 3.3 or ghae-issue-4757 or ghec %}
## Reutilizar flujos de trabajo
{% data reusables.actions.reusable-workflows %}
{% endif %}

## Utilizar ambientes

Puedes configurr ambientes con reglas de protección y secretos. Cad job en un flujo de trabajo puede referenciar un solo ambiente. Cualquier regla de protección que se configure para el ambiente debe pasar antes de que un job que referencia al ambiente se envíe a un ejecutor. Para obtener más información, consulta la sección "[Utilizar ambientes para despliegue](/actions/deployment/using-environments-for-deployment)".

## Utilizar flujos de trabajo iniciales

{% data reusables.actions.workflow-template-overview %}

{% data reusables.repositories.navigate-to-repo %}
{% data reusables.repositories.actions-tab %}
1. Si tu repositorio ya cuenta con flujos de trabajo: En la esquina superior izquierda, da clic sobre **Flujo de trabajo nuevo**. ![Crear un flujo de trabajo nuevo](/assets/images/help/repository/actions-new-workflow.png)
1. Debajo del nombre del flujo de trabajo inicial que te gustaría utilizar, haz clic en **Configurar este flujo de trabajo**. ![Configurar este flujo de trabajo](/assets/images/help/settings/actions-create-starter-workflow.png)

## Pasos siguientes

Para seguir aprendiendo sobre las {% data variables.product.prodname_actions %}, consulta la sección "[Compartir flujos de trabajo, secretos y ejecutores con tu organización](/actions/learn-github-actions/sharing-workflows-secrets-and-runners-with-your-organization)".
