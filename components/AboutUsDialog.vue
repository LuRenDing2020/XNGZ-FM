<template>
  <UiDialog>
    <UiDialogTrigger as-child>
      <slot />
    </UiDialogTrigger>
    <UiDialogContent class="w-[92vw] rounded-md">
      <UiDialogHeader>
        <UiDialogTitle class="text-xl font-bold text-start">
          关于我们
        </UiDialogTitle>
      </UiDialogHeader>
      <div class="flex flex-col my-2">
        <NuxtImg src="/xnhs-logo.svg" class="mx-auto mb-4 w-[35%]" />
        <NuxtImg src="/sys-logo.svg" class="mx-auto mb-4 w-[80%]" />
        <section class="prose mb-4">
          <p class="font-bold">
            系统测试中，遇到问题可联系 QQ 2491076737 进行反馈！
          </p>
        </section>
        <section class="prose">
          <p>
            <b>溪南高中广播站</b>是校团委下与学生会同级的校级组织。该投稿系统由<b>深圳中学 COSMO 智慧校园平台运研中心</b>为<b>风华子衿广播站</b>开发的点歌系统修改而来，并且针对溪南高中广播站的需求进行了部分功能的修改。
          </p>
        </section>
        <NuxtImg src="/cosmo-logo.svg" class="mx-auto my-0 w-[70%]" />
        <section class="prose">
          <p>
            <b>特别感谢 COSMO智慧校园平台运研中心</b>
          </p>
          <p class="mb-4">
            这个项目的代码是开源的，你可以在
            <NuxtLink to="https://github.com/SMS-COSMO/the1068fm" class="font-bold">
              <span>这里</span>
            </NuxtLink>
            查看源代码。
          </p>
          <section class="flex gap-4">
            <NuxtLink to="/manage/login" class="text-gray-600 flex justify-center items-center gap-1 font-serif">
              STAFF通道
            </NuxtLink>
          </section>
        </section>
      </div>
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
