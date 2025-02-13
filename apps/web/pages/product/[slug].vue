<template>
  <NuxtLayout name="default" :breadcrumbs="breadcrumbs">
    <NarrowContainer>
      <div class="md:grid gap-x-6 grid-areas-product-page grid-cols-product-page">
        <section class="grid-in-left-top md:h-full xl:max-h-[700px]">
          <NuxtLazyHydrate when-idle>
            <Gallery :images="addWebpExtensionForSfImages(productGetters.getGallery(product))" />
          </NuxtLazyHydrate>
        </section>
        <section class="mb-10 grid-in-right md:mb-0">
          <NuxtLazyHydrate when-idle>
            <UiPurchaseCard v-if="product" :product="product" :review-average="productReviewAverage" />
          </NuxtLazyHydrate>
        </section>
        <section class="grid-in-left-bottom md:mt-8">
          <UiDivider class="mt-4 mb-2 md:mt-8" />
          <NuxtLazyHydrate when-visible>
            <ProductAccordion v-if="product" :product="product" />
          </NuxtLazyHydrate>
          <NuxtLazyHydrate when-visible>
            <ReviewsAccordion :product="product" :total-reviews="reviewGetters.getTotalReviews(productReviewAverage)" />
          </NuxtLazyHydrate>
        </section>
      </div>
      <section class="mx-4 mt-28 mb-20">
        <NuxtLazyHydrate when-visible>
          <ProductRecommendedProducts :product="product"></ProductRecommendedProducts>
        </NuxtLazyHydrate>
      </section>
    </NarrowContainer>
  </NuxtLayout>
</template>

<script setup lang="ts">
import { Product } from '@plentymarkets/shop-api';
import { productGetters, reviewGetters } from '@plentymarkets/shop-sdk';

const { data: categoryTree } = useCategoryTree();
const { setProductMetaData } = useStructuredData();
const route = useRoute();
const { selectVariation } = useProducts();
const localePath = useLocalePath();
const { addWebpExtensionForSfImages } = useImageUrl();

definePageMeta({
  layout: false,
});

const { productParams, productId } = createProductParams(route.params);
const { data: product, fetchProduct, setTitle, generateBreadcrumbs, breadcrumbs } = useProduct(productId);
const { data: productReviewAverage, fetchProductReviewAverage } = useProductReviewAverage(productId);
const { fetchProductReviews } = useProductReviews(Number(productId));
if (process.server) {
  await Promise.all([
    fetchProduct(productParams),
    fetchProductReviewAverage(Number(productId)),
    fetchProductReviews(Number(productId)),
  ]);
  setProductMetaData(product.value, categoryTree.value[0]);
} else {
  await Promise.all([fetchProduct(productParams), fetchProductReviewAverage(Number(productId))]);
}
selectVariation(productParams.variationId ? product.value : ({} as Product));
setTitle();
generateBreadcrumbs();
// eslint-disable-next-line unicorn/expiring-todo-comments
/* TODO: This should only be temporary.
 *  It changes the url of the product page while on the page and switching the locale.
 *  Should be removed when the item search is refactored.
 */
watch(
  () => product.value.texts.urlPath,
  (value, oldValue) => {
    if (value !== oldValue) {
      navigateTo({
        path: localePath(`/${productGetters.getUrlPath(product.value)}_${productGetters.getItemId(product.value)}`),
        query: route.query,
      });
    }
  },
);
</script>
