<template>
  <UiDialog>
    <UiDialogTrigger as-child>
      <slot />
    </UiDialogTrigger>
    <UiDialogContent class="w-[92vw] rounded-md">
      <UiDialogHeader>
        <UiDialogTitle class="text-xl font-bold text-start">
          规则介绍
        </UiDialogTitle>
      </UiDialogHeader>
      <UiScrollArea class="h-[50vh] sm:h-auto">
        <div class="prose">
          <ol>
            <li>歌曲内容务必积极向上，充满正能量，适合在校园内播放</li>
            <li>与上周歌单有重复的歌曲不播放</li>
            <li>系统正在调试中，目前仅支持 QQ 音乐</li>
            <li>需购买专辑的歌曲不播放</li>
            <li>歌单仅供参考，歌曲播出顺序按实际为准</li>
          </ol>
        </div>
      </UiScrollArea>
      <UiDialogFooter>
        <UiDialogClose as-child>
          <UiButton type="button" variant="secondary" class="ml-auto px-6">
            好的
          </UiButton>
        </UiDialogClose>
      </UiDialogFooter>
    </UiDialogContent>
  </UiDialog>
</template>
