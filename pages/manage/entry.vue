<script setup lang="ts">
import { Loader2 } from 'lucide-vue-next';
import { useForm } from 'vee-validate';
import { toTypedSchema } from '@vee-validate/zod';
import { z } from 'zod';
import { useSongStore } from '~/stores/song';
import type { TSafeSong } from '~/types';
import type { TSong, TSongList } from '~/types';

definePageMeta({
  pageTransition: {
    name: 'slide-down',
    mode: 'out-in',
  },
});

useHead({
  title: '手动录入稿件 · 溪南高中广播站',
  meta: [
    { name: 'description', content: '手动录入稿件 · 溪南高中广播站' },
  ],
});

const songList = ref<TSongList>([]);
const unsetList = computed(
  () => songList.value.filter(s => (s.status === 'unset'))
    .sort((a, b) => (a.createdAt > b.createdAt ? -1 : 1)), // Newest first
);

const isDesktop = ref(true);

const showLength = reactive({
  unset: 100,
  approved: 100,
  rejected: 100,
});

const selectedSong = ref<TSong>();
const searchLoading = ref(false);
const searchListCache = ref<Map<string, any>>(new Map());
const searchList = ref();
async function setSearchList() {
  searchLoading.value = true;
  searchList.value = await getSearchList(selectedSong.value);
  searchLoading.value = false;
}
async function getSearchList(song?: TSong) {
  // Use cache
  const name = `${song?.name ?? ''} ${song?.creator ?? ''}`;
  const mapVal = searchListCache.value.get(name);
  if (mapVal)
    return mapVal;

  // const res: any = (await useFetch('/api/search/song', {
  //   method: 'get',
  //   params: {
  //     key: name,
  //   },
  // }));
  const res: any = (await useFetch('https://428a-218-19-25-63.ngrok-free.app/api/search/song', {
    method: 'get',
    params: {
      key: name,
    },
  }));

  // Store cache
  const data = res.data.value;
  searchListCache.value.set(name, data);
  return data;
};

const listLoading = ref(false);
onMounted(async () => {
  try {
  // @ts-expect-error window
    isDesktop.value = window.innerWidth > 800 && window.innerHeight > 600;
    await $api.user.tokenValidity.query();
  } catch {
    navigateTo('/manage/login');
  }

  try {
    listLoading.value = true;
    songList.value = await $api.song.listUnused.query();
    listLoading.value = false;
  } catch (err) {
    useErrorHandler(err);
  }

  // prefetch songList cache
  for (const song of unsetList.value.slice(0, 100))
    await getSearchList(song);
});

const emit = defineEmits<{ (event: 'submitSuccess', song: TSafeSong): void }>();
const { $toast, $api } = useNuxtApp();

const formSchema = toTypedSchema(z.object({
  name: z.string({ required_error: '歌名长度至少为1' })
    .min(1, '歌名长度至少为1').max(50, '歌名长度最大为50')
    .refine(val => !(val.trim().startsWith('《') || val.trim().endsWith('》')), '歌曲名不需带书名号'),
  creator: z.string({ required_error: '歌手名长度至少为1' })
    .min(1, '歌手名长度至少为1').max(50, '歌手长度最大为50'),
  submitterName: z.string({ required_error: '提交者名字长度至少为2' })
    .min(2, '提交者名字长度至少为2').max(15, '提交者名字长度最大为15'),
  submitterGrade: z.coerce.number({ invalid_type_error: '请填一个数字' })
    .int('请填一个整数').min(1, '年级为1~4').max(4, '年级为1~4'),
  submitterClass: z.coerce.number({ invalid_type_error: '请填一个数字' })
    .min(0, '班级号应大于0').max(100, '班级号应小于100'),
  type: z.enum(
    ['normal', 'withMsg'],
    { errorMap: () => ({ message: '提交了不存在的歌曲类型' }) },
  ),
  message: z.string().nullish(),
}));

const { handleSubmit, values, resetForm } = useForm({
  validationSchema: formSchema,
  initialValues: {
    type: 'normal',
  },
});

const [buttonLoading, toggleLoading] = useToggle(false);
const onSubmit = handleSubmit(async (values) => {
  console.log('开始提交');
  toggleLoading();
  try {
    // 检查提交者是否为自动填充，若为自动填充则忽略错误
    if (values.submitterName === defaultSubmitter.value && values.submitterName.length < 2) {
      return;
    }
    console.log('提交...');
    const id = await $api.song.create.mutate(values);
    const song = {
      id,
      name: values.name,
      creator: values.creator,
      message: values.type === 'withMsg',
      createdAt: new Date(), // fake createdAt because it is assigned on server
    };
    const songStore = useSongStore();
    songStore.submitSong(id);
    const userStore = useUserStore();
    userStore.submit();

    $toast.success('提交成功！');
    resetForm();
    emit('submitSuccess', song);
  } catch (err) {
    useErrorHandler(err);
  }
  toggleLoading();
  console.log('done');
});

// 手动录入稿件页面设置默认提交内容
const defaultSubmitter = ref('管理员手动录入1');// 默认管理员手动录入
const defaultGrade = ref('4'); // 默认年级为其他
const defaultClass = ref(2); // 默认班级为 2 班
</script>

<template>
  <div class="flex flex-col lg:flex-row gap-3 lg:gap-5 lg:h-screen p-4 lg:p-5">
    <UiCard class="basis-1/4 pt-4 relative">
      <UiCardHeader class="px-4 pt-3 lg:px-6 lg:pt-6">
        <UiCardTitle class="flex flex-row">
          <span class="icon-[tabler--square-check] mr-2" />
          已添加
          <UiBadge variant="secondary" class="ml-2 self-center">
            {{ unsetList.length }}
          </UiBadge>
        </UiCardTitle>
      </UiCardHeader>
      <UiCardContent class="px-4 lg:px-6 overflow-hidden">
        <ContentLoading v-if="listLoading" class="lg:h-[calc(100vh-13rem)]" />
        <UiScrollArea v-else class="lg:h-[calc(100vh-13rem)]">
          <TransitionGroup name="list" tag="ul" class="flex flex-row w-max gap-2 lg:block lg:w-full">
            <li v-for="song in unsetList.slice(0, showLength.unset)" :key="song.id">
              <UiContextMenu>
                <UiContextMenuTrigger>
                  <MusicCard
                    :compact="!isDesktop" :song="song" show-grade :selected="selectedSong === song" class="cursor-pointer w-[calc(100vw-6rem)] lg:w-full h-full"
                    @click="selectedSong = song; setSearchList()"
                  />
                </UiContextMenuTrigger>
              </UiContextMenu>
            </li>
          </TransitionGroup>
          <UiAlert v-if="showLength.unset < unsetList.length">
            <UiAlertDescription class="flex flex-row">
              <span class="self-center">
                出于性能考虑，仅加载前 {{ showLength.unset }} 首歌
              </span>
              <UiButton variant="secondary" class="float-right ml-auto" @click="showLength.unset += 50">
                加载更多
              </UiButton>
            </UiAlertDescription>
          </UiAlert>
          <UiScrollBar v-if="!isDesktop" orientation="horizontal" />
        </UiScrollArea>
      </UiCardContent>
    </UiCard>

    <UiCard class="basis-3/4 pt-4">
      <UiCardHeader class="items-start gap-4 space-y-0 flex-row px-4 pt-3 lg:px-6 lg:pt-6">
      <div>
        <UiCardTitle class="flex flex-row">
          <span class="icon-[tabler--square-plus] mr-2" />
          手动录入稿件
        </UiCardTitle>
        <UiCardDescription class="mt-1">
          本页面用于录入投稿箱内收到的 <b class="text-gray-600">纸质投稿</b>，需要投稿请前往 <b><NuxtLink to="/" class="text-gray-600 font-bold">主页</NuxtLink></b> 。
        </UiCardDescription>
      </div>
        <UiButton variant="secondary" class="self-center my-[-10px] ml-auto" @click="navigateTo('/manage')">
          返回排歌模式
        </UiButton>
      </UiCardHeader>
      <UiScrollArea class="lg:h-[calc(100vh-10rem)] px-4 pt-3 lg:px-6 lg:pt-6">
        <form class="grid grid-cols-1 gap-4" @submit="onSubmit">
          <UiFormField v-slot="{ componentField }" name="type">
            <UiFormItem>
              <UiFormLabel>投稿类型</UiFormLabel>
              <UiSelect v-bind="componentField">
                <UiFormControl>
                  <UiSelectTrigger>
                    <UiSelectValue placeholder="请选择一种类型" />
                  </UiSelectTrigger>
                </UiFormControl>
                <UiSelectContent>
                  <UiSelectGroup>
                    <UiSelectItem value="normal">
                      歌曲
                    </UiSelectItem>
                    <UiSelectItem value="withMsg">
                      歌曲+留言稿
                    </UiSelectItem>
                  </UiSelectGroup>
                </UiSelectContent>
              </UiSelect>
              <UiFormMessage />
            </UiFormItem>
          </UiFormField>
          <!-- <UiFormField v-slot="{ componentField }" name="musicapp">
            <UiFormItem>
              <UiFormLabel>音乐来源</UiFormLabel>
              <UiSelect v-bind="componentField">
                <UiFormControl>
                  <UiSelectTrigger>
                    <UiSelectValue placeholder="请选择一个平台" />
                  </UiSelectTrigger>
                </UiFormControl>
                <UiSelectContent>
                  <UiSelectGroup>
                    <UiSelectItem value="1">
                      QQ音乐
                    </UiSelectItem>
                    <UiSelectItem value="1">
                      网易云音乐
                    </UiSelectItem>
                  </UiSelectGroup>
                </UiSelectContent>
              </UiSelect>
              <UiFormMessage />
            </UiFormItem>
          </UiFormField> -->
          <UiFormField v-slot="{ componentField }" name="name">
            <UiFormItem>
              <UiFormLabel>歌名</UiFormLabel>
              <UiFormControl>
                <UiInput type="text" v-bind="componentField" />
              </UiFormControl>
              <UiFormMessage />
            </UiFormItem>
          </UiFormField>

          <UiFormField v-slot="{ componentField }" name="creator">
            <UiFormItem>
              <UiFormLabel>作者</UiFormLabel>
              <UiFormControl>
                <UiInput type="text" v-bind="componentField" />
              </UiFormControl>
              <UiFormMessage />
            </UiFormItem>
          </UiFormField>

          <UiFormField v-if="values.type === 'withMsg'" v-slot="{ componentField }" name="message">
            <UiFormItem>
              <UiFormLabel>投稿留言</UiFormLabel>
              <UiFormControl>
                <UiTextarea v-bind="componentField" />
              </UiFormControl>
              <UiFormMessage />
            </UiFormItem>
          </UiFormField>

          <UiFormField v-slot="{ componentField }" name="submitterName">
            <UiFormItem>
              <UiFormLabel>提交者（删除框内数字“1”以确认）</UiFormLabel>
              <UiFormControl>
                <UiInput type="text" v-bind="componentField" v-model="defaultSubmitter"/>
              </UiFormControl>
              <UiFormMessage />
            </UiFormItem>
          </UiFormField>

          <div class="flex gap-2">
            <UiFormField v-slot="{ componentField }" name="submitterGrade"  v-model="defaultGrade">
              <UiFormItem class="grow">
                <UiFormLabel>年级</UiFormLabel>
                <UiFormControl>
                  <UiSelect v-bind="componentField">
                    <UiFormControl>
                      <UiSelectTrigger>
                        <UiSelectValue placeholder="选择" />
                      </UiSelectTrigger>
                    </UiFormControl>
                    <UiSelectContent>
                      <UiSelectGroup>
                        <UiSelectItem value="1" class="text-sm">
                          高一
                        </UiSelectItem>
                        <UiSelectItem value="2" class="text-sm">
                          高二
                        </UiSelectItem>
                        <UiSelectItem value="3" class="text-sm">
                          高三
                        </UiSelectItem>
                        <UiSelectItem value="4" class="text-sm">
                          其他
                        </UiSelectItem>
                      </UiSelectGroup>
                    </UiSelectContent>
                  </UiSelect>
                </UiFormControl>
                <UiFormMessage />
              </UiFormItem>
            </UiFormField>

            <UiFormField v-slot="{ componentField }" name="submitterClass" class=""  v-model="defaultClass">
              <UiFormItem class="flex flex-col flex-1 basis-2/3">
                <UiFormLabel>班级（请填数字）</UiFormLabel>
                <UiFormControl>
                  <UiInput type="number" v-bind="componentField" />
                </UiFormControl>
                <UiFormMessage />
              </UiFormItem>
            </UiFormField>
          </div>
          <UiDialogFooter>
            <UiButton
              type="submit" class="mt-3 ml-auto px-6 font-bold text-md flex items-center justify-center"
              :disabled="buttonLoading"
            >
              <Loader2 v-show="buttonLoading" class="w-4 h-4 mr-2 animate-spin" />
              录入
            </UiButton>
          </UiDialogFooter>
        </form>
      </UiScrollArea>
      <UiCardContent class="px-4 lg:px-6">

      </UiCardContent>
    </UiCard>
  </div>
</template>
