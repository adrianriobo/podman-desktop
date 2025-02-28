<script lang="ts">
import { onDestroy, onMount } from 'svelte';
import MonacoEditor from '../editor/MonacoEditor.svelte';
import NavPage from '../ui/NavPage.svelte';
import * as jsYaml from 'js-yaml';
import type { V1Route } from '../../../../main/src/plugin/api/openshift-types';
import type { V1NamespaceList } from '@kubernetes/client-node/dist/api';
import ErrorMessage from '../ui/ErrorMessage.svelte';

export let resourceId: string;
export let engineId: string;

let kubeDetails: string;

let defaultContextName: string;
let currentNamespace: string;
let allNamespaces: V1NamespaceList;
let deployStarted = false;
let deployFinished = false;
let deployError = '';
let updatePodInterval: NodeJS.Timeout;
let openshiftConsoleURL: string;

let deployUsingServices = true;
let deployUsingRoutes = true;
let createdPod = undefined;
let bodyPod;

let createdRoutes: V1Route[] = [];

onMount(async () => {
  // grab kube result from the pod
  const kubeResult = await window.generatePodmanKube(engineId, [resourceId]);
  kubeDetails = kubeResult;

  // parse yaml
  bodyPod = jsYaml.load(kubeDetails) as any;

  // grab default context
  defaultContextName = await window.kubernetesGetCurrentContextName();

  // grab current namespace
  currentNamespace = await window.kubernetesGetCurrentNamespace();

  // check that the variable is set to a value, otherwise set to default namespace
  if (!currentNamespace) {
    currentNamespace = 'default';
  }

  // grab all the namespaces (will be useful to provide a drop-down to select the namespace)
  try {
    allNamespaces = await window.kubernetesListNamespaces();
  } catch (error) {
    console.debug('Not able to list all namespaces, probably a permission error', error);
  }

  // check if there is OpenShift and then grab openshift console URL
  try {
    const openshiftConfigMap = await window.kubernetesReadNamespacedConfigMap(
      'console-public',
      'openshift-config-managed',
    );
    openshiftConsoleURL = openshiftConfigMap?.data?.consoleURL;
  } catch (error) {
    // probably not OpenShift cluster, ignoring
    console.debug('Probably not an OpenShift cluster, so ignoring the error to grab console link', error);
  }
});

function openOpenshiftConsole() {
  // build link to openOpenshiftConsole
  const linkToOpen = `${openshiftConsoleURL}/k8s/ns/${currentNamespace}/pods/${createdPod.metadata.name}`;
  window.openExternal(linkToOpen);
}

async function updatePod() {
  const updatedPod = await window.kubernetesReadNamespacedPod(createdPod.metadata.name, createdPod.metadata.namespace);
  createdPod = updatedPod;
  // stop to monitor once it is running
  if (createdPod.status?.phase === 'Running') {
    clearInterval(updatePodInterval);
    deployFinished = true;
    window.telemetryTrack('deployToKube.running', {
      useServices: deployUsingServices,
      useRoutes: deployUsingRoutes,
    });
  }
}

onDestroy(() => {
  // reset any timeout
  clearInterval(updatePodInterval);
});

function goBackToHistory(): void {
  window.history.go(-1);
}

function openRoute(route: V1Route) {
  window.openExternal(`http://${route.spec.host}`);
}

async function deployToKube() {
  deployStarted = true;
  deployFinished = false;
  deployError = '';
  createdPod = undefined;
  // reset any timeout
  clearInterval(updatePodInterval);

  createdRoutes = [];
  let servicesToCreate: any[] = [];
  let routesToCreate: any[] = [];

  if (bodyPod?.spec?.volumes) {
    delete bodyPod.spec.volumes;
  }

  // if we deploy using services, we need to get rid of .hostPort and generate kubernetes services object
  if (deployUsingServices) {
    // collect all ports
    bodyPod.spec?.containers?.forEach((container: any) => {
      if (container.volumeMounts) {
        delete container.volumeMounts;
      }

      container?.ports?.forEach((port: any) => {
        if (port.hostPort) {
          // create service
          const service = {
            apiVersion: 'v1',
            kind: 'Service',
            metadata: {
              name: `${bodyPod.metadata.name}-${port.hostPort}`,
              namespace: currentNamespace,
            },
            spec: {
              ports: [
                {
                  name: port.name,
                  port: port.hostPort,
                  protocol: port.protocol || 'TCP',
                  targetPort: port.containerPort,
                },
              ],
              selector: {
                app: bodyPod.metadata.name,
              },
            },
          };
          servicesToCreate.push(service);

          if (openshiftConsoleURL && deployUsingRoutes) {
            // Create OpenShift route object
            const route = {
              apiVersion: 'route.openshift.io/v1',
              kind: 'Route',
              metadata: {
                name: `${bodyPod.metadata.name}-${port.hostPort}`,
                namespace: currentNamespace,
              },
              spec: {
                port: {
                  targetPort: port.containerPort,
                },
                to: {
                  kind: 'Service',
                  name: `${bodyPod.metadata.name}-${port.hostPort}`,
                },
              },
            };
            routesToCreate.push(route);
          }
          // delete
          delete port.hostPort;
        }
      });
    });
  }

  // https://github.com/kubernetes-client/javascript/issues/487
  if (bodyPod?.metadata?.creationTimestamp) {
    bodyPod.metadata.creationTimestamp = new Date(bodyPod.metadata.creationTimestamp);
  }

  const eventProperties = {
    useServices: deployUsingServices,
    useRoutes: deployUsingRoutes,
  };
  if (openshiftConsoleURL) {
    eventProperties['isOpenshift'] = true;
  }

  try {
    createdPod = await window.kubernetesCreatePod(currentNamespace, bodyPod);

    // create services
    for (const service of servicesToCreate) {
      await window.kubernetesCreateService(currentNamespace, service);
    }

    // Create routes
    for (const route of routesToCreate) {
      const createdRoute = await window.openshiftCreateRoute(currentNamespace, route);
      createdRoutes = [...createdRoutes, createdRoute];
    }
    window.telemetryTrack('deployToKube', eventProperties);

    // update status
    updatePodInterval = setInterval(updatePod, 2000);
  } catch (error) {
    window.telemetryTrack('deployToKube', { ...eventProperties, errorMessage: error.message });
    deployError = error;
    deployStarted = false;
    deployFinished = false;
  }
}

$: bodyPod && updateKubeResult();

function updateKubeResult() {
  kubeDetails = jsYaml.dump(bodyPod, { noArrayIndent: true, quotingType: '"', lineWidth: -1 });
}
</script>

<NavPage title="Deploy generated pod to Kubernetes" searchEnabled="{false}">
  <div slot="empty" class="p-5 bg-zinc-700 h-full">
    <div class="bg-zinc-800 h-full p-5">
      {#if kubeDetails}
        <p>Generated pod to deploy to Kubernetes:</p>
        <div class="h-1/3 pt-2">
          <MonacoEditor content="{kubeDetails}" language="yaml" />
        </div>
      {/if}

      {#if bodyPod}
        <div class="pt-2 pb-4">
          <label for="contextToUse" class="block mb-1 text-sm font-medium text-gray-300">Pod Name:</label>
          <input
            type="text"
            bind:value="{bodyPod.metadata.name}"
            name="podName"
            id="podName"
            class=" cursor-default w-full p-2 outline-none text-sm bg-zinc-900 rounded-sm text-gray-400 placeholder-gray-400"
            required />
        </div>
      {/if}

      <div class="pt-2 m-2">
        <label for="services" class="block mb-1 text-sm font-medium text-gray-300">Use Kubernetes Services:</label>
        <input
          type="checkbox"
          bind:checked="{deployUsingServices}"
          name="useServices"
          id="useServices"
          class=""
          required />
        <span class="text-gray-300 text-sm ml-1"
          >Replace .hostPort exposure on containers by Services. It is the recommended way to expose ports, as a cluster
          policy may prevent to use hostPort.</span>
      </div>

      <!-- Allow to create routes for OpenShift clusters -->
      {#if openshiftConsoleURL}
        <div class="pt-2 m-2">
          <label for="routes" class="block mb-1 text-sm font-medium text-gray-300">Create OpenShift routes:</label>
          <input type="checkbox" bind:checked="{deployUsingRoutes}" name="useRoutes" id="useRoutes" class="" required />
          <span class="text-gray-300 text-sm ml-1"
            >Create OpenShift routes to get access to the exposed ports of this pod.</span>
        </div>
      {/if}

      {#if defaultContextName}
        <div class="pt-2">
          <label for="contextToUse" class="block mb-1 text-sm font-medium text-gray-300">Kubernetes Context:</label>
          <input
            type="text"
            bind:value="{defaultContextName}"
            name="defaultContextName"
            id="defaultContextName"
            readonly
            class="cursor-default w-full p-2 outline-none text-sm bg-zinc-900 rounded-sm text-gray-400 placeholder-gray-400"
            required />
        </div>
      {/if}

      {#if allNamespaces}
        <div class="pt-2">
          <label for="namespaceToUse" class="block mb-1 text-sm font-medium text-gray-300">Kubernetes namespace:</label>
          <select
            class="w-full p-2 outline-none text-sm bg-zinc-900 rounded-sm text-gray-400 placeholder-gray-400"
            name="namespaceChoice"
            bind:value="{currentNamespace}">
            {#each allNamespaces.items as namespace}
              <option value="{namespace.metadata.name}">
                {namespace.metadata.name}
              </option>
            {/each}
          </select>
        </div>
      {/if}

      {#if !deployStarted}
        <div class="pt-2 m-2">
          <button
            on:click="{() => deployToKube()}"
            class="w-full pf-c-button pf-m-primary"
            type="button"
            disabled="{bodyPod?.metadata?.name === ''}">
            <span class="pf-c-button__icon pf-m-start">
              <i class="fas fa-rocket" aria-hidden="true"></i>
            </span>
            Deploy
          </button>
        </div>
      {/if}
      <ErrorMessage class="text-sm" error="{deployError}" />

      {#if createdPod}
        <div class="bg-zinc-900 p-5 my-4">
          <div class="flex flex-row items-center">
            <div>Created pod:</div>
            {#if openshiftConsoleURL && createdPod?.metadata?.name}
              <div class="justify-end flex flex-1">
                <div class="pf-c-button pf-m-link cursor-pointer" on:click="{() => openOpenshiftConsole()}">
                  <span class="pf-c-button__icon pf-m-start">
                    <i class="fas fa-external-link-alt" aria-hidden="true"></i>
                  </span>
                  Open in OpenShift console
                </div>
              </div>
            {/if}
          </div>
          <div class="text-gray-400">
            {#if createdPod.metadata?.name}
              <p class="pt-2">Name: {createdPod.metadata.name}</p>
            {/if}
            {#if createdPod.status?.phase}
              <p class="pt-2">Phase: {createdPod.status.phase}</p>
            {/if}

            {#if createdPod.status?.containerStatuses}
              <p class="pt-2">Container statuses:</p>
              <ul class="list-disc list-inside">
                {#each createdPod.status.containerStatuses as containerStatus}
                  <li class="pt-2">
                    {containerStatus.name}
                    {#if containerStatus.ready}
                      <span class="text-gray-500">Ready</span>
                    {/if}
                    {#if containerStatus.state?.running}
                      <span class="text-green-400">(Running)</span>
                    {/if}
                    {#if containerStatus.state?.terminated}
                      <span class="text-red-500">(Terminated)</span>
                    {/if}
                    {#if containerStatus.state?.waiting}
                      <span class="text-yellow-500">(Waiting)</span>
                      {#if containerStatus.state.waiting.reason}
                        <span class="text-yellow-500">[{containerStatus.state.waiting.reason}]</span>
                      {/if}
                    {/if}
                  </li>
                {/each}
              </ul>
            {/if}
            {#if createdRoutes && createdRoutes.length > 0}
              <p class="pt-2">Endpoints:</p>
              <ul class="list-disc list-inside">
                {#each createdRoutes as createdRoute}
                  <li class="pt-2">
                    Port {createdRoute.spec.port.targetPort} is reachable with route
                    <span
                      class="cursor-pointer text-violet-400 hover:text-violet-600 hover:no-underline"
                      on:click="{() => {
                        openRoute(createdRoute);
                      }}">{createdRoute.metadata.name}</span>
                  </li>
                {/each}
              </ul>
            {/if}
          </div>

          <!-- add editor for the result-->
          <div class="h-[100px] pt-2">
            <MonacoEditor content="{jsYaml.dump(createdPod)}" language="yaml" />
          </div>
        </div>
      {/if}

      {#if deployFinished}
        <button on:click="{() => goBackToHistory()}" class="pt-4 w-full pf-c-button pf-m-primary">Done</button>
      {/if}
    </div>
  </div>
</NavPage>
