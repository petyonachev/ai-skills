# Forms and validation

Input is validated **at the boundary**, before it reaches a service — that is the
one rule everything here serves (`engineering-standards`, `security`). There are
two idioms: **request DTOs** for APIs (the modern default) and **Forms** for
server-rendered HTML. Pick by surface, not habit.

---

## Request DTOs — the API idiom

Map the request body straight onto a typed, constrained DTO and let the framework
validate it. `#[MapRequestPayload]` deserializes and validates in one step,
returning `422` automatically on failure — no manual validation in the controller.

```php
final readonly class CreateOrderRequest
{
    public function __construct(
        #[Assert\NotBlank]
        public string $reference,

        #[Assert\Count(min: 1)]
        #[Assert\Valid]
        /** @var list<OrderLineRequest> */
        public array $lines,

        #[Assert\Positive]
        public int $customerId,
    ) {}
}

#[Route('/orders', methods: ['POST'])]
public function create(
    #[MapRequestPayload] CreateOrderRequest $request,
): JsonResponse {
    $order = $this->placeOrder->handle($request); // service gets a valid DTO
    return $this->json($order, Response::HTTP_CREATED);
}
```

Use `#[MapQueryString]` for GET query parameters, and `#[Assert\Valid]` to cascade
validation into nested DTOs. The service layer receives an already-valid object
and never re-checks structural validity.

---

## The Validator

Constraints are attributes on the DTO/entity properties. The common set:
`NotBlank`, `NotNull`, `Length`, `Range`, `Positive`, `Email`, `Choice`, `Count`,
`Valid` (cascade), `Type`. Prefer them over hand-written checks.

**Validation groups** let one object validate differently by context (e.g.
`Default` vs `strict`):

```php
#[Assert\NotBlank(groups: ['registration'])]
public string $password;
// validate($dto, groups: ['registration'])
```

**Custom constraints** for real domain rules that constraints cannot express — a
`Constraint` + a `ConstraintValidator`. Keep them about *validity of input*, not
business decisions.

---

## Forms — the HTML idiom

For server-rendered pages and admin screens, use the Form component. Bind the form
to a DTO (preferred) or an entity via `data_class`, then `handleRequest` and check
`isSubmitted()/isValid()`:

```php
final class OrderType extends AbstractType
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefault('data_class', CreateOrderRequest::class);
    }
    // buildForm(): ->add('reference', TextType::class) ...
}

$form = $this->createForm(OrderType::class);
$form->handleRequest($request);
if ($form->isSubmitted() && $form->isValid()) {
    $this->placeOrder->handle($form->getData());
}
```

Prefer binding a form to a **DTO** over binding directly to a Doctrine entity —
binding to entities lets a malformed submission mutate a managed object before
validation.

---

## Where validation belongs

- **Boundary (DTO/Form + Validator)** — structural and format validity of incoming
  data. This is the first line and the one users see.
- **Domain (entity invariants)** — the entity's constructor/methods still enforce
  what must always be true, because data reaches persistence through paths a form
  never touches (`database-design` pushes the last line into DB constraints).

Do not rely on form validation alone for domain invariants — the boundary can be
bypassed; the domain and database cannot.

---

## Pitfalls

- **Manual validation in the controller** — reading raw request data and checking
  it by hand instead of a DTO + `#[MapRequestPayload]`.
- **Business rules as form constraints** — "customer must have credit" is a domain
  decision, not an input constraint.
- **Trusting client-side validation** — always revalidate server-side (`security`).
- **Binding forms straight to entities** — malformed input mutating a managed
  entity; bind to a DTO.
- **Skipping `#[Assert\Valid]`** on nested DTOs, so nested payloads go unchecked.
- **Duplicating boundary checks in the service** — the service should trust it
  received a valid DTO; enforce domain invariants, not structural ones.
