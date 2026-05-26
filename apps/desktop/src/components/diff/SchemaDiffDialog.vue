<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { useI18n } from "vue-i18n";
import { Dialog, DialogHeader, DialogTitle, DialogFooter, DialogContent } from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { Label } from "@/components/ui/label";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { useConnectionStore } from "@/stores/connectionStore";
import DatabaseIcon from "@/components/icons/DatabaseIcon.vue";
import * as api from "@/lib/api";
import { isSchemaAware } from "@/lib/databaseCapabilities";
import {
  diffColumns,
  diffForeignKeys,
  diffIndexes,
  diffTables,
  diffTriggers,
  generateSyncSql,
  type TableDiff,
} from "@/lib/schemaDiff";
import { sqlMetadataRefreshTarget } from "@/lib/sqlMetadataRefresh";
import { useToast } from "@/composables/useToast";
import { Loader2, Copy, Play, GitCompareArrows, ArrowLeftRight, ChevronDown, ChevronRight } from "lucide-vue-next";

interface SelectableTableDiff extends TableDiff {
  selected: boolean;
}

const { t } = useI18n();
const { toast } = useToast();
const open = defineModel<boolean>("open", { default: false });
const store = useConnectionStore();

const props = defineProps<{
  prefillConnectionId?: string;
  prefillDatabase?: string;
  prefillSchema?: string;
}>();

const STORAGE_KEY = "dbx-schema-diff-last";

function loadLastSelections() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : null;
  } catch {
    return null;
  }
}

function saveLastSelections() {
  try {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({
        sourceConnectionId: sourceConnectionId.value,
        sourceDatabase: sourceDatabase.value,
        sourceSchema: sourceSchema.value,
        targetConnectionId: targetConnectionId.value,
        targetDatabase: targetDatabase.value,
        targetSchema: targetSchema.value,
      }),
    );
  } catch {}
}

const last = loadLastSelections();
const sourceConnectionId = ref(last?.sourceConnectionId ?? "");
const sourceDatabase = ref(last?.sourceDatabase ?? "");
const sourceDatabases = ref<string[]>([]);
const sourceSchema = ref(last?.sourceSchema ?? "");
const sourceSchemas = ref<string[]>([]);

const targetConnectionId = ref(last?.targetConnectionId ?? "");
const targetDatabase = ref(last?.targetDatabase ?? "");
const targetDatabases = ref<string[]>([]);
const targetSchema = ref(last?.targetSchema ?? "");
const targetSchemas = ref<string[]>([]);

const step = ref<"select" | "comparing" | "result" | "syncResult">("select");
const diffs = ref<SelectableTableDiff[]>([]);
const loadingMeta = ref(false);
const executing = ref(false);
const executedCount = ref(0);
const executeTotal = ref(0);
const syncErrors = ref<{ sql: string; error: string }[]>([]);
const syncSuccessCount = ref(0);

const allSelected = computed(() => diffs.value.length > 0 && diffs.value.every((d) => d.selected));
const someSelected = computed(() => diffs.value.some((d) => d.selected) && !allSelected.value);

function groupDiffs(type: "added" | "removed" | "modified") {
  return diffs.value.filter((d) => d.type === type);
}

function isGroupAllSelected(type: "added" | "removed" | "modified") {
  const group = groupDiffs(type);
  return group.length > 0 && group.every((d) => d.selected);
}

function isGroupSomeSelected(type: "added" | "removed" | "modified") {
  const group = groupDiffs(type);
  return group.some((d) => d.selected) && !isGroupAllSelected(type);
}

function toggleGroup(type: "added" | "removed" | "modified") {
  const next = !isGroupAllSelected(type);
  diffs.value.filter((d) => d.type === type).forEach((d) => (d.selected = next));
}
function toggleAll() {
  const next = !allSelected.value;
  diffs.value.forEach((d) => (d.selected = next));
}

const groupCollapsed = ref<Record<string, boolean>>({});

function isCollapsed(type: string) {
  return groupCollapsed.value[type] ?? false;
}

function toggleCollapse(type: string) {
  groupCollapsed.value = { ...groupCollapsed.value, [type]: !isCollapsed(type) };
}

function groupSelectedCount(type: "added" | "removed" | "modified") {
  return groupDiffs(type).filter((d) => d.selected).length;
}

const groupOrder = [
  { type: "modified" as const, iconColor: "text-blue-500", icon: "~" },
  { type: "added" as const, iconColor: "text-green-500", icon: "+" },
  { type: "removed" as const, iconColor: "text-red-500", icon: "\u2716" },
];

const syncSql = computed(() => {
  const selected = diffs.value.filter((d) => d.selected);
  if (selected.length === 0) return "";
  const srcConfig = store.getConfig(targetConnectionId.value);
  return generateSyncSql(selected, srcConfig?.db_type || "mysql", targetSchema.value);
});

const sqlConnections = computed(() =>
  store.connections.filter((c) => !["redis", "mongodb", "elasticsearch"].includes(c.db_type)),
);

const canCompare = computed(
  () =>
    sourceConnectionId.value &&
    sourceDatabase.value &&
    sourceSchema.value &&
    targetConnectionId.value &&
    targetDatabase.value &&
    targetSchema.value,
);

function connectionIconType(connectionId: string) {
  const config = store.getConfig(connectionId);
  return config?.driver_profile || config?.db_type || "mysql";
}

function swapSourceTarget() {
  const tmpConnId = sourceConnectionId.value;
  const tmpDb = sourceDatabase.value;
  const tmpDbs = sourceDatabases.value;
  const tmpSchema = sourceSchema.value;
  const tmpSchemas = sourceSchemas.value;
  sourceConnectionId.value = targetConnectionId.value;
  sourceDatabase.value = targetDatabase.value;
  sourceDatabases.value = targetDatabases.value;
  sourceSchema.value = targetSchema.value;
  sourceSchemas.value = targetSchemas.value;
  targetConnectionId.value = tmpConnId;
  targetDatabase.value = tmpDb;
  targetDatabases.value = tmpDbs;
  targetSchema.value = tmpSchema;
  targetSchemas.value = tmpSchemas;
  resetResult();
}

async function loadDatabases(connectionId: string, side: "source" | "target") {
  if (!connectionId) return;
  loadingMeta.value = true;
  try {
    await store.ensureConnected(connectionId);
    const dbs = await api.listDatabases(connectionId);
    const names = dbs.map((d) => d.name);
    if (side === "source") {
      sourceDatabases.value = names;
      sourceDatabase.value = names.length === 1 ? names[0] : "";
      sourceSchemas.value = [];
      sourceSchema.value = "";
    } else {
      targetDatabases.value = names;
      targetDatabase.value = names.length === 1 ? names[0] : "";
      targetSchemas.value = [];
      targetSchema.value = "";
    }
  } catch {
    if (side === "source") sourceDatabases.value = [];
    else targetDatabases.value = [];
  } finally {
    loadingMeta.value = false;
  }
}

async function loadSchemas(side: "source" | "target", preferredSchema = "") {
  const connectionId = side === "source" ? sourceConnectionId.value : targetConnectionId.value;
  const database = side === "source" ? sourceDatabase.value : targetDatabase.value;
  if (!connectionId || !database) return;
  const config = store.getConfig(connectionId);
  if (!isSchemaAware(config?.db_type)) {
    if (side === "source") {
      sourceSchemas.value = [];
      sourceSchema.value = database;
    } else {
      targetSchemas.value = [];
      targetSchema.value = database;
    }
    return;
  }

  const schemas = await api.listSchemas(connectionId, database);
  const selected =
    preferredSchema && schemas.includes(preferredSchema)
      ? preferredSchema
      : schemas.includes("public")
        ? "public"
        : (schemas[0] ?? "");
  if (side === "source") {
    sourceSchemas.value = schemas;
    sourceSchema.value = selected;
  } else {
    targetSchemas.value = schemas;
    targetSchema.value = selected;
  }
}

async function startCompare() {
  if (!canCompare.value) return;
  step.value = "comparing";
  diffs.value = [];
  groupCollapsed.value = {};

  try {
    await store.ensureConnected(sourceConnectionId.value);
    await store.ensureConnected(targetConnectionId.value);

    const srcTables = await api.listTables(sourceConnectionId.value, sourceDatabase.value, sourceSchema.value);
    const tgtTables = await api.listTables(targetConnectionId.value, targetDatabase.value, targetSchema.value);

    const srcTableNames = srcTables.filter((t) => t.table_type !== "VIEW").map((t) => t.name);
    const tgtTableNames = tgtTables.filter((t) => t.table_type !== "VIEW").map((t) => t.name);
    const srcTableComments = new Map(srcTables.map((t) => [t.name, t.comment ?? null]));
    const tgtTableComments = new Map(tgtTables.map((t) => [t.name, t.comment ?? null]));
    const srcViewNames = srcTables.filter((t) => t.table_type === "VIEW").map((t) => t.name);
    const tgtViewNames = tgtTables.filter((t) => t.table_type === "VIEW").map((t) => t.name);
    const { added, removed, common } = diffTables(srcTableNames, tgtTableNames);
    const { added: addedViews, removed: removedViews } = diffTables(srcViewNames, tgtViewNames);

    const result: SelectableTableDiff[] = [];

    for (const name of added) {
      const ddl = await api.getTableDdl(sourceConnectionId.value, sourceDatabase.value, sourceSchema.value, name);
      result.push({ type: "added", objectType: "table", name, ddl, selected: false });
    }

    for (const name of removed) {
      result.push({ type: "removed", objectType: "table", name, selected: false });
    }

    for (const name of addedViews) {
      result.push({ type: "added", objectType: "view", name, selected: false });
    }

    for (const name of removedViews) {
      result.push({ type: "removed", objectType: "view", name, selected: false });
    }

    for (const name of common) {
      const [srcCols, tgtCols, srcIdx, tgtIdx, srcFks, tgtFks, srcTriggers, tgtTriggers] = await Promise.all([
        api.getColumns(sourceConnectionId.value, sourceDatabase.value, sourceSchema.value, name),
        api.getColumns(targetConnectionId.value, targetDatabase.value, targetSchema.value, name),
        api.listIndexes(sourceConnectionId.value, sourceDatabase.value, sourceSchema.value, name),
        api.listIndexes(targetConnectionId.value, targetDatabase.value, targetSchema.value, name),
        api.listForeignKeys(sourceConnectionId.value, sourceDatabase.value, sourceSchema.value, name),
        api.listForeignKeys(targetConnectionId.value, targetDatabase.value, targetSchema.value, name),
        api.listTriggers(sourceConnectionId.value, sourceDatabase.value, sourceSchema.value, name),
        api.listTriggers(targetConnectionId.value, targetDatabase.value, targetSchema.value, name),
      ]);

      const colDiffs = diffColumns(srcCols, tgtCols);
      const idxDiffs = diffIndexes(srcIdx, tgtIdx);
      const fkDiffs = diffForeignKeys(srcFks, tgtFks);
      const triggerDiffs = diffTriggers(srcTriggers, tgtTriggers);
      const srcComment = srcTableComments.get(name) ?? null;
      const tgtComment = tgtTableComments.get(name) ?? null;
      const commentChanged = (srcComment ?? "") !== (tgtComment ?? "");

      if (
        colDiffs.length > 0 ||
        idxDiffs.length > 0 ||
        fkDiffs.length > 0 ||
        triggerDiffs.length > 0 ||
        commentChanged
      ) {
        result.push({
          type: "modified",
          objectType: "table",
          name,
          columns: colDiffs.length > 0 ? colDiffs : undefined,
          indexes: idxDiffs.length > 0 ? idxDiffs : undefined,
          foreignKeys: fkDiffs.length > 0 ? fkDiffs : undefined,
          triggers: triggerDiffs.length > 0 ? triggerDiffs : undefined,
          sourceTableComment: commentChanged ? srcComment : undefined,
          targetTableComment: commentChanged ? tgtComment : undefined,
          selected: false,
        });
      }
    }

    diffs.value = result;
    step.value = "result";
  } catch (e: any) {
    toast(e?.message || String(e), 5000);
    step.value = "select";
  }
}

async function executeSql() {
  const sql = syncSql.value.trim();
  if (!sql || executing.value) return;
  executing.value = true;
  syncErrors.value = [];

  const statements = sql
    .split(/;\s*\n/)
    .map((s) => s.trim())
    .map((s) =>
      s
        .split("\n")
        .filter((line) => !line.trimStart().startsWith("--"))
        .join("\n")
        .trim(),
    )
    .filter((s) => s && !s.startsWith("--"));
  executeTotal.value = statements.length;
  executedCount.value = 0;

  try {
    await store.ensureConnected(targetConnectionId.value);
    for (const stmt of statements) {
      try {
        await api.executeQuery(targetConnectionId.value, targetDatabase.value, stmt, targetSchema.value);
      } catch (e: any) {
        syncErrors.value.push({ sql: stmt, error: e?.message || String(e) });
      }
      executedCount.value++;
    }
    const failed = syncErrors.value.length;
    syncSuccessCount.value = statements.length - failed;
    const refreshTarget = sqlMetadataRefreshTarget(sql, targetSchema.value);
    if (refreshTarget.scope === "connection") {
      await store.loadDatabases(targetConnectionId.value, { force: true });
    } else if (refreshTarget.scope === "database") {
      await store.refreshObjectListTreeNode(targetConnectionId.value, targetDatabase.value, refreshTarget.schema);
    }
    step.value = "syncResult";
  } catch (e: any) {
    toast(e?.message || String(e), 5000);
  } finally {
    executing.value = false;
  }
}

function copySql() {
  navigator.clipboard.writeText(syncSql.value);
  toast(t("grid.copied"));
}

function diffBadgeVariant(type: string) {
  if (type === "added") return "default";
  if (type === "removed") return "destructive";
  return "secondary";
}

function diffLabel(type: string) {
  if (type === "added") return t("diff.added");
  if (type === "removed") return t("diff.removed");
  return t("diff.modified");
}

function resetResult() {
  step.value = "select";
  diffs.value = [];
  syncErrors.value = [];
  syncSuccessCount.value = 0;
  executedCount.value = 0;
  executeTotal.value = 0;
  groupCollapsed.value = {};
}

watch(sourceConnectionId, (id) => {
  sourceDatabase.value = "";
  loadDatabases(id, "source");
  resetResult();
});

watch(targetConnectionId, (id) => {
  targetDatabase.value = "";
  loadDatabases(id, "target");
  resetResult();
});

watch(sourceDatabase, (database) => {
  sourceSchema.value = "";
  sourceSchemas.value = [];
  resetResult();
  if (database) loadSchemas("source", props.prefillSchema).catch((e) => toast(String(e), 5000));
});
watch(targetDatabase, (database) => {
  targetSchema.value = "";
  targetSchemas.value = [];
  resetResult();
  if (database) loadSchemas("target").catch((e) => toast(String(e), 5000));
});
watch(sourceSchema, () => resetResult());
watch(targetSchema, () => resetResult());

watch([sourceConnectionId, sourceDatabase, sourceSchema, targetConnectionId, targetDatabase, targetSchema], () =>
  saveLastSelections(),
);

watch(
  open,
  async (val) => {
    if (val) {
      step.value = "select";
      diffs.value = [];
      syncErrors.value = [];
      executedCount.value = 0;
      executeTotal.value = 0;
      if (props.prefillConnectionId) {
        sourceConnectionId.value = props.prefillConnectionId;
        await loadDatabases(props.prefillConnectionId, "source");
        if (props.prefillDatabase) {
          sourceDatabase.value = props.prefillDatabase;
          await loadSchemas("source", props.prefillSchema);
        }
      } else {
        const saved = loadLastSelections();
        if (saved) {
          if (saved.sourceConnectionId) {
            await loadDatabases(saved.sourceConnectionId, "source");
            if (saved.sourceDatabase) {
              sourceDatabase.value = saved.sourceDatabase;
              await loadSchemas("source", saved.sourceSchema);
            }
          }
          if (saved.targetConnectionId) {
            await loadDatabases(saved.targetConnectionId, "target");
            if (saved.targetDatabase) {
              targetDatabase.value = saved.targetDatabase;
              await loadSchemas("target", saved.targetSchema);
            }
          }
        }
      }
    }
  },
  { immediate: true },
);
</script>

<template>
  <Dialog v-model:open="open">
    <DialogContent class="sm:max-w-2xl max-h-[80vh] flex flex-col overflow-hidden" @interact-outside.prevent>
      <DialogHeader>
        <DialogTitle class="flex items-center gap-2">
          <GitCompareArrows class="w-4 h-4" />
          {{ t("diff.title") }}
        </DialogTitle>
      </DialogHeader>

      <div class="flex-1 min-h-0 overflow-auto space-y-4 py-2">
        <!-- Source / Target Selection -->
        <div class="grid grid-cols-[1fr_auto_1fr] gap-4 items-start">
          <div class="space-y-2">
            <Label class="text-xs font-medium">{{ t("diff.source") }}</Label>
            <Select
              :model-value="sourceConnectionId"
              @update:model-value="(v: any) => (sourceConnectionId = String(v))"
            >
              <SelectTrigger class="h-8 text-xs">
                <div class="flex items-center gap-2">
                  <DatabaseIcon
                    v-if="sourceConnectionId"
                    :db-type="connectionIconType(sourceConnectionId)"
                    class="w-3.5 h-3.5"
                  />
                  <SelectValue :placeholder="t('diff.selectConnection')" />
                </div>
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="c in sqlConnections" :key="c.id" :value="c.id">
                  <div class="flex items-center gap-2">
                    <DatabaseIcon :db-type="c.driver_profile || c.db_type" class="w-3.5 h-3.5" />
                    {{ c.name }}
                  </div>
                </SelectItem>
              </SelectContent>
            </Select>
            <Select
              v-if="sourceDatabases.length"
              :model-value="sourceDatabase"
              @update:model-value="(v: any) => (sourceDatabase = String(v))"
            >
              <SelectTrigger class="h-8 text-xs">
                <SelectValue :placeholder="t('diff.selectDatabase')" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="db in sourceDatabases" :key="db" :value="db">{{ db }}</SelectItem>
              </SelectContent>
            </Select>
            <Select
              v-if="sourceSchemas.length"
              :model-value="sourceSchema"
              @update:model-value="(v: any) => (sourceSchema = String(v))"
            >
              <SelectTrigger class="h-8 text-xs">
                <SelectValue :placeholder="t('diff.selectSchema')" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="schema in sourceSchemas" :key="schema" :value="schema">{{ schema }}</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <div class="flex items-center pt-6">
            <Button variant="ghost" size="icon" class="h-7 w-7" :title="t('diff.swap')" @click="swapSourceTarget">
              <ArrowLeftRight class="w-3.5 h-3.5" />
            </Button>
          </div>

          <div class="space-y-2">
            <Label class="text-xs font-medium">{{ t("diff.target") }}</Label>
            <Select
              :model-value="targetConnectionId"
              @update:model-value="(v: any) => (targetConnectionId = String(v))"
            >
              <SelectTrigger class="h-8 text-xs">
                <div class="flex items-center gap-2">
                  <DatabaseIcon
                    v-if="targetConnectionId"
                    :db-type="connectionIconType(targetConnectionId)"
                    class="w-3.5 h-3.5"
                  />
                  <SelectValue :placeholder="t('diff.selectConnection')" />
                </div>
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="c in sqlConnections" :key="c.id" :value="c.id">
                  <div class="flex items-center gap-2">
                    <DatabaseIcon :db-type="c.driver_profile || c.db_type" class="w-3.5 h-3.5" />
                    {{ c.name }}
                  </div>
                </SelectItem>
              </SelectContent>
            </Select>
            <Select
              v-if="targetDatabases.length"
              :model-value="targetDatabase"
              @update:model-value="(v: any) => (targetDatabase = String(v))"
            >
              <SelectTrigger class="h-8 text-xs">
                <SelectValue :placeholder="t('diff.selectDatabase')" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="db in targetDatabases" :key="db" :value="db">{{ db }}</SelectItem>
              </SelectContent>
            </Select>
            <Select
              v-if="targetSchemas.length"
              :model-value="targetSchema"
              @update:model-value="(v: any) => (targetSchema = String(v))"
            >
              <SelectTrigger class="h-8 text-xs">
                <SelectValue :placeholder="t('diff.selectSchema')" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="schema in targetSchemas" :key="schema" :value="schema">{{ schema }}</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </div>

        <Button v-if="step === 'select'" size="sm" :disabled="!canCompare || loadingMeta" @click="startCompare">
          <Loader2 v-if="loadingMeta" class="w-3.5 h-3.5 mr-1 animate-spin" />
          <GitCompareArrows v-else class="w-3.5 h-3.5 mr-1" />
          {{ loadingMeta ? t("common.loading") : t("diff.compare") }}
        </Button>

        <!-- Comparing -->
        <div v-if="step === 'comparing'" class="flex items-center justify-center py-8 text-sm text-muted-foreground">
          <Loader2 class="w-4 h-4 animate-spin mr-2" />
          {{ t("diff.comparing") }}
        </div>

        <!-- Results -->
        <template v-if="step === 'result'">
          <div v-if="diffs.length === 0" class="py-6 text-center text-sm text-muted-foreground">
            {{ t("diff.noDifferences") }}
          </div>

          <template v-else>
            <div class="border rounded-lg overflow-hidden text-xs">
              <!-- Select All -->
              <div class="flex items-center gap-2 px-3 py-2 bg-muted/50 border-b">
                <input
                  :checked="allSelected"
                  :indeterminate="someSelected"
                  class="accent-primary"
                  type="checkbox"
                  @change="toggleAll"
                />
                <span class="font-medium">{{ t("diff.selectAll") }}</span>
                <span class="text-muted-foreground">({{ diffs.length }})</span>
              </div>

              <!-- Groups -->
              <template v-for="g in groupOrder" :key="g.type">
                <div v-if="groupDiffs(g.type).length > 0">
                  <!-- Group Header -->
                  <div
                    :class="isCollapsed(g.type) ? 'border-b' : 'bg-muted/30'"
                    class="flex items-center gap-2 px-3 py-2 border-t border-b cursor-pointer hover:bg-accent/30 select-none"
                    @click="toggleCollapse(g.type)"
                  >
                    <button class="text-muted-foreground" @click.stop="toggleCollapse(g.type)">
                      <ChevronDown v-if="!isCollapsed(g.type)" class="w-3.5 h-3.5" />
                      <ChevronRight v-else class="w-3.5 h-3.5" />
                    </button>
                    <input
                      :checked="isGroupAllSelected(g.type)"
                      :indeterminate="isGroupSomeSelected(g.type)"
                      class="accent-primary"
                      type="checkbox"
                      @change="toggleGroup(g.type)"
                      @click.stop
                    />
                    <span :class="g.iconColor" class="font-bold text-sm">{{ g.icon }}</span>
                    <span class="font-medium">{{ diffLabel(g.type) }}</span>
                    <span class="text-muted-foreground">
                      ({{
                        t("diff.groupSummary", {
                          selected: groupSelectedCount(g.type),
                          total: groupDiffs(g.type).length,
                        })
                      }})
                    </span>
                  </div>

                  <!-- Group Items -->
                  <div v-if="!isCollapsed(g.type)">
                    <div
                      v-for="d in groupDiffs(g.type)"
                      :key="d.name"
                      class="flex items-center gap-2 px-3 py-1.5 border-t border-border/50 hover:bg-accent/20"
                    >
                      <input v-model="d.selected" class="accent-primary ml-7" type="checkbox" />
                      <span class="font-mono truncate w-1/3">{{ d.name }}</span>
                      <span class="flex-1 text-muted-foreground truncate">
                        <template v-if="d.type === 'modified' && d.columns">
                          <span v-for="(col, ci) in d.columns" :key="`col-${ci}`">
                            <span
                              :class="{
                                'text-green-500': col.type === 'added',
                                'text-red-500': col.type === 'removed',
                                'text-yellow-500': col.type === 'modified',
                              }"
                              >{{ col.type === "added" ? "+" : col.type === "removed" ? "-" : "~" }}{{ col.name }}</span
                            >
                            <span v-if="ci < d.columns!.length - 1">, </span>
                          </span>
                        </template>
                        <template v-if="d.type === 'modified' && d.indexes">
                          <span v-if="d.columns?.length">; </span>
                          {{ t("diff.indexes") }}:
                          <span v-for="(idx, ii) in d.indexes" :key="`idx-${ii}`">
                            <span
                              :class="{
                                'text-green-500': idx.type === 'added',
                                'text-red-500': idx.type === 'removed',
                                'text-yellow-500': idx.type === 'modified',
                              }"
                              >{{ idx.type === "added" ? "+" : idx.type === "removed" ? "-" : "~" }}{{ idx.name }}</span
                            >
                            <span v-if="ii < d.indexes!.length - 1">, </span>
                          </span>
                        </template>
                        <template v-if="d.type === 'modified' && d.foreignKeys">
                          <span v-if="d.columns?.length || d.indexes?.length">; </span>
                          {{ t("diff.foreignKeys") }}:
                          <span v-for="(fk, fi) in d.foreignKeys" :key="`fk-${fi}`">
                            <span
                              :class="{
                                'text-green-500': fk.type === 'added',
                                'text-red-500': fk.type === 'removed',
                                'text-yellow-500': fk.type === 'modified',
                              }"
                              >{{ fk.type === "added" ? "+" : fk.type === "removed" ? "-" : "~" }}{{ fk.name }}</span
                            >
                            <span v-if="fi < d.foreignKeys!.length - 1">, </span>
                          </span>
                        </template>
                        <template v-if="d.type === 'modified' && d.triggers">
                          <span v-if="d.columns?.length || d.indexes?.length || d.foreignKeys?.length">; </span>
                          {{ t("diff.triggers") }}:
                          <span v-for="(trigger, ti) in d.triggers" :key="`trigger-${ti}`">
                            <span
                              :class="{
                                'text-green-500': trigger.type === 'added',
                                'text-red-500': trigger.type === 'removed',
                                'text-yellow-500': trigger.type === 'modified',
                              }"
                              >{{ trigger.type === "added" ? "+" : trigger.type === "removed" ? "-" : "~"
                              }}{{ trigger.name }}</span
                            >
                            <span v-if="ti < d.triggers!.length - 1">, </span>
                          </span>
                        </template>
                        <span v-if="d.type === 'added' && !d.columns" class="text-green-500">{{
                          t("diff.newTable")
                        }}</span>
                        <span v-if="d.type === 'removed' && !d.columns" class="text-red-500">{{
                          t("diff.dropTable")
                        }}</span>
                      </span>
                    </div>
                  </div>
                </div>
              </template>
            </div>

            <!-- SQL Preview -->
            <div class="space-y-1">
              <Label class="text-xs font-medium">{{ t("diff.generatedSql") }}</Label>
              <textarea
                :value="syncSql"
                readonly
                class="w-full h-48 rounded-lg border bg-muted/20 p-3 font-mono text-xs resize-none focus:outline-none focus:ring-1 focus:ring-ring"
              />
            </div>

            <!-- Sync Errors -->
            <div v-if="syncErrors.length > 0" class="space-y-1">
              <Label class="text-xs font-medium text-destructive">
                {{ t("diff.syncSummary", { success: executeTotal - syncErrors.length, failed: syncErrors.length }) }}
              </Label>
              <div class="max-h-32 overflow-auto border rounded-lg bg-destructive/5 p-2 space-y-1">
                <div v-for="(err, i) in syncErrors" :key="i" class="text-xs font-mono">
                  <span class="text-destructive">{{ err.error }}</span>
                  <span class="text-muted-foreground ml-1"
                    >— {{ err.sql.slice(0, 80) }}{{ err.sql.length > 80 ? "..." : "" }}</span
                  >
                </div>
              </div>
            </div>
          </template>
        </template>

        <!-- Sync Result -->
        <template v-if="step === 'syncResult'">
          <div class="space-y-4 py-4">
            <div class="flex items-center gap-3">
              <div
                :class="syncErrors.length === 0 ? 'bg-green-100 text-green-600' : 'bg-yellow-100 text-yellow-600'"
                class="w-10 h-10 rounded-full flex items-center justify-center"
              >
                <span v-if="syncErrors.length === 0" class="text-lg">&#10003;</span>
                <span v-else class="text-lg">!</span>
              </div>
              <div>
                <div class="text-sm font-medium">
                  {{ syncErrors.length === 0 ? t("diff.syncSuccess") : t("diff.syncPartial") }}
                </div>
                <div class="text-xs text-muted-foreground">
                  {{ t("diff.syncResultSummary", { success: syncSuccessCount, failed: syncErrors.length }) }}
                </div>
              </div>
            </div>

            <div v-if="syncErrors.length > 0" class="space-y-1">
              <Label class="text-xs font-medium text-destructive">{{ t("diff.syncErrors") }}</Label>
              <div class="max-h-48 overflow-auto border rounded-lg bg-destructive/5 p-2 space-y-1">
                <div v-for="(err, i) in syncErrors" :key="i" class="text-xs font-mono">
                  <span class="text-destructive">{{ err.error }}</span>
                  <span class="text-muted-foreground ml-1"
                    >— {{ err.sql.slice(0, 100) }}{{ err.sql.length > 100 ? "..." : "" }}</span
                  >
                </div>
              </div>
            </div>
          </div>
        </template>
      </div>

      <DialogFooter v-if="step === 'result' && diffs.length > 0" class="flex items-center gap-2">
        <span v-if="executing" class="text-xs text-muted-foreground mr-auto">
          {{ t("diff.syncProgress", { current: executedCount, total: executeTotal }) }}
        </span>
        <Button variant="outline" size="sm" @click="copySql">
          <Copy class="w-3 h-3 mr-1" /> {{ t("diff.copySql") }}
        </Button>
        <Button size="sm" :disabled="!syncSql.trim() || executing" @click="executeSql">
          <Loader2 v-if="executing" class="w-3 h-3 animate-spin mr-1" />
          <Play v-else class="w-3 h-3 mr-1" />
          {{ t("diff.executeSync") }}
        </Button>
      </DialogFooter>

      <DialogFooter v-if="step === 'syncResult'" class="flex items-center gap-2">
        <Button size="sm" variant="outline" @click="startCompare">
          <GitCompareArrows class="w-3 h-3 mr-1" />
          {{ t("diff.reCompare") }}
        </Button>
        <Button size="sm" @click="open = false">{{ t("common.close") }}</Button>
      </DialogFooter>
    </DialogContent>
  </Dialog>
</template>
