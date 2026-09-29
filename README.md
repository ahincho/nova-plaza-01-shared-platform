# Plaza

**Plaza es una plataforma de compras construida sobre [Nova](https://github.com/ahincho/nova-shared-01-docs)**,
con un servicio en cada framework que Nova soporta: NestJS, Spring Boot y Quarkus. Existe para
mostrar en un sistema de verdad lo que la plataforma promete: que los tres stacks se ven y se
comportan igual, que una compra deja una sola traza aunque cruce tres frameworks, y que cada
capacidad de Nova tiene un consumidor real.

La decisión está en [ADR-043](https://github.com/ahincho/nova-shared-01-docs/blob/main/adrs/shared/ADR-043-plaza-la-plataforma-de-compras.md).

## Los servicios

| Servicio | Framework | Repositorio | Es dueño de | Puerto local |
|---|---|---|---|---|
| `plaza-bff` | NestJS | `nova-plaza-02-nestjs-bff` | la experiencia del cliente y la compra | 8080 |
| `plaza-orders` | Spring Boot | `nova-plaza-03-spring-boot-orders` | los pedidos y su estado | 8081 |
| `plaza-catalog` | Quarkus | `nova-plaza-04-quarkus-catalog` | los productos, los precios y el stock | 8082 |

Este repositorio es la entrada al producto: la descripción, el compose que levanta lo que los
servicios necesitan y el guion de la demo. El puerto 3000 queda libre para Grafana.

```mermaid
flowchart LR
    client([cliente]) --> bff[plaza-bff<br/>NestJS]
    bff -->|reservar y confirmar stock| catalog[plaza-catalog<br/>Quarkus]
    bff -->|crear y confirmar el pedido| orders[plaza-orders<br/>Spring Boot]
    catalog --> catalogdb[(Postgres<br/>catalog)]
    orders --> ordersdb[(Postgres<br/>orders)]
    orders -. credenciales .-> vault[(Vault)]
    catalog -. credenciales .-> vault
```

## La compra

El BFF orquesta la compra y es el único que habla con los dos servicios; los servicios de Java no
se llaman entre sí.

1. Reserva el stock en el catálogo, que devuelve los precios del momento.
2. Crea el pedido, en estado `PENDING`, con esos precios.
3. Confirma la reserva.
4. Confirma el pedido.

Si un paso falla, el BFF deshace los anteriores: libera la reserva y cancela el pedido. Nada se
reintenta, y una compensación que se pierde no bloquea stock, porque la reserva vence sola a los
diez minutos. La compra lleva un `Idempotency-Key`, así que el cliente puede repetirla sin comprar
dos veces.

## Levantarlo en local

Requiere Docker con Compose.

```bash
docker compose up -d --wait
```

Levanta una base Postgres por servicio y Vault en modo dev, con un secreto por servicio ya cargado:

| Secreto en Vault | Claves |
|---|---|
| `secret/plaza/orders/db` | `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` |
| `secret/plaza/catalog/db` | `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` |

Vault escucha en `http://localhost:8200`, y su token está en `.env.example`. Los valores son solo
para esta máquina; para cambiarlos se copia `.env.example` a `.env`.

Las trazas, los logs y las métricas van al stack de observabilidad de
[`nova-shared-03-infrastructure`](https://github.com/ahincho/nova-shared-03-infrastructure), que se
levanta aparte.

Para bajarlo todo:

```bash
docker compose down -v
```

## Estado

| Fase | Contenido | Estado |
|---|---|---|
| 1 | este repositorio, el catálogo y los pedidos, con sus secretos en Vault | en curso |
| 2 | el BFF y la compra orquestada | pendiente |
| 3 | la mensajería con Kafka y el ranking de lo más vendido | pendiente |
| 4 | CQRS en los pedidos | pendiente |
| 5 | un servicio nuevo generado desde el arquetipo | pendiente |

## Licencia

[Eclipse Public License 2.0](LICENSE).
