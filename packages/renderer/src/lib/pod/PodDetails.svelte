<script lang="ts">
import Route from '../../Route.svelte';
import { onDestroy, onMount } from 'svelte';
import type { PodInfoUI } from './PodInfoUI';
import { PodUtils } from './pod-utils';
import type { Unsubscriber } from 'svelte/store';
import { podsInfos } from '../../stores/pods';
import PodIcon from '../images/PodIcon.svelte';
import StatusIcon from '../images/StatusIcon.svelte';
import PodActions from './PodActions.svelte';
import PodDetailsSummary from './PodDetailsSummary.svelte';
import PodDetailsInspect from './PodDetailsInspect.svelte';
import PodDetailsKube from './PodDetailsKube.svelte';
import PodDetailsLogs from './PodDetailsLogs.svelte';
import { faExclamationCircle } from '@fortawesome/free-solid-svg-icons';
import Tooltip from '../ui/Tooltip.svelte';
import Fa from 'svelte-fa/src/fa.svelte';
import DetailsTab from '../ui/DetailsTab.svelte';
import ErrorMessage from '../ui/ErrorMessage.svelte';

export let podName: string;
export let engineId: string;
export let kind: string;

let pod: PodInfoUI;
let podUnsubscribe: Unsubscriber;

onMount(() => {
  const podUtils = new PodUtils();
  // loading volume info
  podUnsubscribe = podsInfos.subscribe(pods => {
    const matchingPod = pods.find(
      podInPods => podInPods.Name === podName && podInPods.engineId === engineId && kind === podInPods.kind,
    );
    if (matchingPod) {
      try {
        pod = podUtils.getPodInfoUI(matchingPod);
      } catch (err) {
        console.error(err);
      }
    }
  });
});

onDestroy(() => {
  if (podUnsubscribe) {
    podUnsubscribe();
  }
});

function inProgressCallback(inProgress: boolean, state: string): void {
  pod.actionInProgress = inProgress;
  if (state && inProgress) {
    pod.status = 'STARTING';
  }
}

function errorCallback(errorMessage: string): void {
  pod.actionError = errorMessage;
  pod.status = 'ERROR';
}
</script>

{#if pod}
  <Route path="/*">
    <div class="w-full h-full">
      <div class="flex h-full flex-col">
        <div class="flex w-full flex-row">
          <div class="w-full px-5 pt-5">
            <div class="flex flew-row items-center">
              <a class="text-violet-400 text-base hover:no-underline" href="/pods" title="Go back to pods list">Pods</a>
              <div class="text-xl mx-2 text-gray-400">></div>
              <div class="text-sm font-extralight text-gray-400">Pod Details</div>
            </div>
            <div class="text-lg flex flex-row items-start pt-1">
              <div class="pr-3 pt-1">
                <StatusIcon icon="{PodIcon}" status="{pod.status}" />
              </div>
              <div class="text-lg flex flex-col">
                <div class="mr-2">{pod.name}</div>
                <div class="mr-2 pb-4 text-small text-gray-500">{pod.shortId}</div>
              </div>
            </div>
            <section class="pf-c-page__main-tabs pf-m-limit-width">
              <div class="pf-c-page__main-body">
                <div class="pf-c-tabs pf-m-page-insets" id="open-tabs-example-tabs-list">
                  <ul class="pf-c-tabs__list">
                    <DetailsTab title="Summary" url="summary" />
                    <DetailsTab title="Logs" url="logs" />
                    <DetailsTab title="Inspect" url="inspect" />
                    <DetailsTab title="Kube" url="kube" />
                  </ul>
                </div>
              </div>
            </section>
          </div>
          <div class="flex flex-col px-5 pt-5">
            <div class="flex justify-end">
              <div class="flex items-center w-5">
                {#if pod.actionError}
                  <ErrorMessage error="{pod.actionError}" icon />
                {:else}
                  <div>&nbsp;</div>
                {/if}
              </div>
              <PodActions
                pod="{pod}"
                inProgressCallback="{(flag, state) => inProgressCallback(flag, state)}"
                errorCallback="{error => errorCallback(error)}"
                detailed="{true}" />
            </div>
          </div>
          <a href="/containers" title="Close Details" class="mt-2 mr-2 text-gray-500"
            ><i class="fas fa-times" aria-hidden="true"></i></a>
        </div>
        <Route path="/summary" breadcrumb="Summary">
          <PodDetailsSummary pod="{pod}" />
        </Route>
        <Route path="/logs" breadcrumb="Logs">
          <PodDetailsLogs pod="{pod}" />
        </Route>
        <Route path="/inspect" breadcrumb="Inspect">
          <PodDetailsInspect pod="{pod}" />
        </Route>
        <Route path="/kube" breadcrumb="Kube">
          <PodDetailsKube pod="{pod}" />
        </Route>
      </div>
    </div>
  </Route>
{/if}
