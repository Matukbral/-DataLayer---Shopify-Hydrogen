# Documentación de DataLayer - Shopify Hydrogen (GA4 / GTM)

Este repositorio contiene la arquitectura, especificación técnica y el diccionario de datos del **dataLayer** implementado en la tienda **Shopify Hydrogen** (Remix framework). 

La implementación está diseñada para capturar la actividad del usuario en el Front-End desacoplado (Headless) y enviar los datos de comercio electrónico estándar a **Google Tag Manager (GTM)** y **Google Analytics 4 (GA4)** utilizando el protocolo de [Shopify Analytics / Web Pixels](https://shopify.dev/docs/api/pixels/custom-pixels) o mediante eventos de Hydration del cliente.

---

## 1. Arquitectura y Configuración General

### Inicialización en el Root (`root.tsx`)
Para evitar la pérdida de eventos durante la navegación SPA (Single Page Application) de Hydrogen, el dataLayer se inicializa de forma global en el `<head>` antes de cargar cualquier script de terceros:

```typescript
// app/root.tsx o entry.client.tsx
import {useEffect} from 'react';
import {useLocation} from '@remix-run/react';

export function useAnalytics() {
  const location = useLocation();

  useEffect(() => {
    // Inicializar dataLayer si no existe
    window.dataLayer = window.dataLayer || [];
    
    // Virtual Page View para SPA
    window.dataLayer.push({
      event: 'virtual_page_view',
      page_path: location.pathname,
      page_title: document.title,
      page_search: location.search,
    });
  }, [location]);
}
```

---

## 2. Diccionario de Eventos de E-commerce

Todos los eventos de comercio electrónico siguen estrictamente el esquema requerido por **GA4**.

### Tabla Resumen de Eventos

| Evento GTM / GA4 | Acción del Usuario | Componente / Trigger | Sincronización |
| :--- | :--- | :--- | :--- |
| `view_item_list` | Ve una colección o lista de productos | Componente `ProductGrid` | Client-Side (In-view) |
| `view_item` | Ve la página de detalle de producto (PDP) | Ruta `/products/$handle` | Server/Client Hydration |
| `add_to_cart` | Añade un producto al carrito de compras | Botón `AddToCartButton` | Client-Side (Fetchers) |
| `remove_from_cart`| Elimina un producto del carrito | Botón `CartLineRemove` | Client-Side (Fetchers) |
| `view_cart` | Abre el drawer o visita la página del carrito | Ruta `/cart` o Drawer Open | Client-Side |
| `begin_checkout` | Hace clic para proceder al checkout de Shopify | Formulario `CartCheckout` | Client-Side (Redirect) |

---

## 3. Especificación Detallada de Carga Útil (Payloads)

### 3.1. Ver Detalle de Producto (`view_item`)
Se dispara cuando el usuario carga la página de un producto específico.

* **Trigger:** Componente de ruta de producto al montarse.
* **Estructura del Objeto:**

```json
{
  "event": "view_item",
  "ecommerce": {
    "currency": "USD",
    "value": 85.00,
    "items": [
      {
        "item_id": "gid://shopify/ProductVariant/456789123",
        "item_name": "Polera de Algodón Orgánico",
        "index": 0,
        "item_brand": "Mi Marca Headless",
        "item_category": "Ropa",
        "item_category_2": "Poleras",
        "price": 85.00,
        "quantity": 1,
        "item_variant": "M / Blanco"
      }
    ]
  }
}
```

### 3.2. Añadir al Carrito (`add_to_cart`)
Se dispara inmediatamente después de una mutación exitosa de la API de la Cart de Shopify (Storefront API).

* **Trigger:** Respuesta `action` exitosa en la mutación `cartAdd`.
* **Estructura del Objeto:**

```json
{
  "event": "add_to_cart",
  "ecommerce": {
    "currency": "USD",
    "value": 170.00,
    "items": [
      {
        "item_id": "gid://shopify/ProductVariant/456789123",
        "item_name": "Polera de Algodón Orgánico",
        "price": 85.00,
        "quantity": 2,
        "item_variant": "M / Blanco"
      }
    ]
  }
}
```

### 3.3. Inicio de Checkout (`begin_checkout`)
Se dispara cuando el usuario hace clic en el botón de pagar y es redirigido al Checkout estándar de Shopify.

* **Trigger:** Intercepción del evento `submit` en el formulario de Checkout hacia la URL externa de Shopify.
* **Estructura del Objeto:**

```json
{
  "event": "begin_checkout",
  "ecommerce": {
    "currency": "USD",
    "value": 255.00,
    "items": [
      {
        "item_id": "gid://shopify/ProductVariant/456789123",
        "item_name": "Polera de Algodón Orgánico",
        "price": 85.00,
        "quantity": 3
      }
    ]
  }
}
```

---

## 4. Tipos de Datos y Validación (TypeScript)

Para mantener la consistencia en el proyecto Hydrogen, se recomienda definir los siguientes tipos para los payloads del dataLayer:

```typescript
export interface DataLayerItem {
  item_id: string;
  item_name: string;
  price: number;
  quantity: number;
  item_brand?: string;
  item_category?: string;
  item_variant?: string;
  index?: number;
}

export interface EcommercePayload {
  currency: string;
  value: number;
  items: DataLayerItem[];
}

export interface DataLayerEvent {
  event: string;
  ecommerce?: EcommercePayload;
  [key: string]: any;
}
```

---

## 5. Pruebas y Validación en Producción

1. **Google Tag Assistant:** Conectar la URL local (`http://localhost:3000`) o de staging para verificar que los eventos se disparen en orden cronológico.
2. **Consola del Navegador:** Escribir `dataLayer` en las herramientas de desarrollador para auditar el historial de objetos persistidos en la SPA.
3. **Validación de Checkout:** Los eventos posteriores a `begin_checkout` (`add_shipping_info`, `purchase`) ocurren en el dominio nativo de Shopify (`checkout.shopify.com`), por lo que deben configurarse mediante **Shopify Web Pixels** usando el mismo formato de eventos en el panel de administración de Shopify.
