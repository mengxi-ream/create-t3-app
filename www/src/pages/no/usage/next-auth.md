---
title: NextAuth.js
description: Bruk av NextAuth.js
layout: ../../../layouts/docs.astro
lang: no
---

Hvis du vil ha et autentiseringssystem i Next.js-applikasjonen din, er NextAuth.js en utmerket løsning for å unngå kompleksiteten med å bygge det selv. Den kommer med en omfattende liste leverandører for raskt å legge til OAuth-autentisering og tilbyr adaptere for mange databaser og ORM-er.

## Kontekstleverandør

I applikasjonens inngangspunkt vil du se at applikasjonen er pakket inn av en [SessionProvider](https://next-auth.js.org/getting-started/client#sessionprovider).

```tsx:pages/_app.tsx
<SessionProvider session={session}>
  <Component {...pageProps} />
</SessionProvider>
```

Denne kontekstleverandøren lar applikasjonen din få tilgang til din `session`-data fra hvor som helst i applikasjonen din uten å måtte sende dem som _props_:

```tsx:pages/users/[id].tsx
import { useSession } from "next-auth/react";

const User = () => {
  const { data: session } = useSession();

  if (!session) {
    // Håndter uautentisert state f.eks. ved å vise en påloggingskomponent
    return <SignIn />;
  }

  return <p>Welcome {session.user.name}!</p>;
};
```

## Henting av session på serversiden

Noen ganger vil du kanskje be om _session_ på serveren. For å gjøre dette, prefetch'er du session ved å bruke `getServerAuthSession`-hjelperfunksjonen som `create-t3-app` gir, sender den videre til klienten ved å bruke `getServerSideProps`:

```tsx:pages/users/[id].tsx
import { getServerAuthSession } from "../server/auth";
import type { GetServerSideProps } from "next";
export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const session = await getServerAuthSession(ctx);
  return {
    props: { session },
  };
};
const User = () => {
  const { data: session } = useSession();
  // MERK: `session` vil ikke ha en lastestatus siden den allerede er prefetched på serveren
  ...
}
```

## Inkluder `user.id` i din Session

`create-t3-app` er konfigurert til å bruke [session callback](https://next-auth.js.org/configuration/callbacks#session-callback) i NextAuth.js-konfigurasjonen for å inkludere bruker-ID i 'session'-objektet.

```ts:server/auth.ts
callbacks: {
    session({ session, user }) {
      if (session.user) {
        session.user.id = user.id;
      }
      return session;
    },
  },
```

Dette er kombinert med en typedeklarasjonsfil for å sikre at `user.id` er riktig typet når du får tilgang til `session`-objektet. Les mer om [`"Module Augmentation"`](https://next-auth.js.org/getting-started/typescript#module-augmentation) i NextAuth.js-dokumentasjonen.

```ts:server/auth.ts
import { DefaultSession } from "next-auth";

declare module "next-auth" {
  interface Session {
    user?: {
      id: string;
    } & DefaultSession["user"];
  }
}
```

Det samme mønsteret kan brukes til å legge til flere data til `session`-objektet, som f.eks et `role`-felt, men **skal ikke misbrukes til å lagre sensitive data på klienten**.

## Bruk med tRPC

Hvis du bruker NextAuth.js med tRPC, kan du opprette gjenbrukbare beskyttede prosedyrer med [middlewares](https://trpc.io/docs/v10/middlewares). Dette lar deg lage prosedyrer som bare er tilgjengelige for autentiserte brukere. `create-t3-app` gir deg allerede dette, slik at du enkelt kan få tilgang til session-objektet i autentiserte prosedyrer.

Dette skjer i to trinn:

1. Få tilgang til sesjonen fra _request-headerne_ ved å bruke funksjonen [`getServerSession`](https://next-auth.js.org/configuration/nextjs#getServerSession). Fordelen med `getServerSession` sammenlignet med `getSession` er at det er en funksjon på serversiden og medfører ikke unødvendige kall. `create-t3-app` lager en hjelpefunksjon som abstraherer dette særegne API-et, slik at du ikke trenger å importere både NextAuth.js-alternativene dine så vel som `getServerSession`-funksjonen hver gang du trenger tilgang til sesssion.

```ts:server/auth.ts
export const getServerAuthSession = (ctx: {
  req: GetServerSidePropsContext["req"];
  res: GetServerSidePropsContext["res"];
}) => {
  return getServerSession(ctx.req, ctx.res, authOptions);
};
```

Med denne hjelpefunksjonen kan vi hente sesjonen og sende den videre til tRPC-konteksten:

```ts:server/api/trpc.ts
import { getServerAuthSession } from "../auth";

export const createContext = async (opts: CreateNextContextOptions) => {
  const { req, res } = opts;
  const session = await getServerAuthSession({ req, res });
  return await createContextInner({
    session,
  });
};
```

2. Lag en tRPC-middleware som sjekker om brukeren er autentisert. Vi bruker deretter middlewaren i en `protectedProcedure`. Hvert kall av disse prosedyrene må autentiseres, ellers kastes en feilmelding, som kan håndteres av klienten.

```ts:server/api/trpc.ts
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({
    ctx: {
      // inferer `session` som ikke-nullbar
      session: { ...ctx.session, user: ctx.session.user },
    },
  });
}));
```

`Session`-objektet er en minimal representasjon av brukeren og inneholder bare noen få felt. Hvis du bruker `protectedProcedures`, har du tilgang til brukerens ID, som kan brukes til å hente ut mer data fra databasen.

```ts:server/api/routers/user.ts
const userRouter = router({
  me: protectedProcedure.query(async ({ ctx }) => {
    const user = await prisma.user.findUnique({
      where: {
        id: ctx.session.user.id,
      },
    });
    return user;
  }),
});
```

## Bruk med Prisma

Mye [førstegangsoppsett](https://authjs.dev/reference/adapter/prisma/) kreves for å bruke NextAuth.js med Prisma. `create-t3-app` vil gjøre dette for deg, og hvis du velger både Prisma og NextAuth.js, får du et fullt funksjonelt autentiseringssystem med alle nødvendige modeller forhåndskonfigurert. Vi oppretter applikasjonen din med en forhåndskonfigurert Discord OAuth-leverandør, som vi valgte siden den er en av de enkleste leverandørene å komme i gang med – du trenger bare å legge inn tokens i `.env`-filen og du er i gang. Du kan imidlertid enkelt legge til flere leverandører ved å følge [NextAuth.js-dokumentasjonen](https://next-auth.js.org/providers/). Vær oppmerksom på at enkelte leverandører krever at du legger til flere felt i enkelte modeller. Vi anbefaler å lese dokumentasjonen for leverandøren du planlegger å bruke for å sikre at alle obligatoriske felt er til stede.

### Legge til nye felt i modellene dine

Hvis du legger til nye felt i noen av modellene `User`, `Account`, `Session` eller `VerificationToken` (du trenger sannsynligvis bare å justere `User`-modellen), må du huske på at [Prisma-adapteren](https://next-auth.js.org/adapters/prisma) automatisk legger til felt i disse modellene når nye brukere registrerer seg og logger på. Så når du legger til nye felt i disse modellene, må du oppgi standardverdier for dem fordi adapteren ikke vet om disse feltene.

Hvis du for eksempel vil legge til et `role`-felt i `User`-modellen, må du angi en standardverdi for feltet. Dette oppnås ved å legge til en `@default`-verdi i `role`-feltet i `User`-modellen:

```diff:prisma/schema.prisma
+ enum Role {
+   USER
+   ADMIN
+ }

  model User {
    ...
+   role Role @default(USER)
  }
```

## Bruk med Next.js middleware

Bruk av NextAuth.js med Next.js middleware [krever bruk av "JWT session strategy"](https://next-auth.js.org/configuration/nextjs#caveats) for autentisering. Dette er fordi middlewaren bare kan få tilgang til session-informasjonskapslene når den er en JWT. Som standard er `create-t3-app` konfigurert til å bruke **default**-databasestrategien, i kombinasjon med Prisma som databaseadapter.

## Oppsett av DiscordProvider (standard)

1. Naviger til [Applications-delen i Discord Developer Portal](https://discord.com/developers/applications) og klikk på "New Application"

2. Bytt til "OAuth2 => Generelt" i settings-menyen

- Kopier klient-ID-en og lim den inn i `AUTH_DISCORD_ID` i `.env`.
- Under Client Secret, klikk på "Reset Secret" og kopier denne strengen til `AUTH_DISCORD_SECRET` i `.env`. Vær forsiktig siden du ikke lenger vil kunne se denne hemmeligheten og tilbakestilling av den vil føre til at den eksisterende hemmeligheten utløper.
- Klikk på "Add Redirect" og lim inn `<app url>/api/auth/callback/discord` (eksempel for utvikling i lokal miljø: <code class="break-all">http://localhost:3000/api/auth/callback/discord</code>)
- Lagre endringene dine
- Det er mulig, men ikke anbefalt, å bruke samme Discord-applikasjon for utvikling og produksjon. Du kan også vurdere å [Mocke leverandøren](https://github.com/trpc/trpc/blob/main/examples/next-prisma-websockets-starter/src/pages/api/auth/%5B...nextauth%5D.ts) under utviklingen.

## Nyttige Ressurser

| Ressurser                        | Link                                    |
| -------------------------------- | --------------------------------------- |
| NextAuth.js Dokumentasjon        | https://next-auth.js.org/               |
| NextAuth.js GitHub               | https://github.com/nextauthjs/next-auth |
| tRPC Kitchen Sink - med NextAuth | https://kitchen-sink.trpc.io/next-auth  |
