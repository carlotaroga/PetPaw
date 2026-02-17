# Configuración de Supabase para PetPaw

## 📋 Resumen

Este documento proporciona ejemplos de uso del cliente Supabase en la aplicación PetPaw.

## 🔐 Autenticación

### Registro de Usuario

```typescript
import { supabase } from "@/lib/supabase";

async function signUp(email: string, password: string, nombre: string) {
  const { data, error } = await supabase.auth.signUp({
    email,
    password,
    options: {
      data: {
        nombre,
      },
    },
  });

  if (error) {
    console.error("Error en registro:", error.message);
    return null;
  }

  return data.user;
}
```

### Inicio de Sesión

```typescript
async function signIn(email: string, password: string) {
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) {
    console.error("Error en login:", error.message);
    return null;
  }

  return data.user;
}
```

### Cerrar Sesión

```typescript
async function signOut() {
  const { error } = await supabase.auth.signOut();
  if (error) console.error("Error al cerrar sesión:", error.message);
}
```

### Obtener Usuario Actual

```typescript
async function getCurrentUser() {
  const {
    data: { user },
  } = await supabase.auth.getUser();
  return user;
}
```

## 🐾 Operaciones con Mascotas

### Listar Todas las Mascotas

```typescript
import { supabase } from "@/lib/supabase";
import type { Tables } from "@/types/database.types";

type Pet = Tables<"pets">;

async function getAllPets(): Promise<Pet[]> {
  const { data, error } = await supabase
    .from("pets")
    .select("*")
    .order("created_at", { ascending: false });

  if (error) {
    console.error("Error al obtener mascotas:", error.message);
    return [];
  }

  return data;
}
```

### Filtrar Mascotas por Categoría

```typescript
async function getPetsByCategory(
  categoria: "Perros" | "Gatos" | "Otros",
): Promise<Pet[]> {
  const { data, error } = await supabase
    .from("pets")
    .select("*")
    .eq("categoria", categoria)
    .eq("estado", "Disponible");

  if (error) {
    console.error("Error al filtrar mascotas:", error.message);
    return [];
  }

  return data;
}
```

### Obtener Detalles de una Mascota

```typescript
async function getPetById(id: string): Promise<Pet | null> {
  const { data, error } = await supabase
    .from("pets")
    .select("*")
    .eq("id", id)
    .single();

  if (error) {
    console.error("Error al obtener mascota:", error.message);
    return null;
  }

  return data;
}
```

### Crear Mascota (Solo Admin)

```typescript
import type { TablesInsert } from "@/types/database.types";

type PetInsert = TablesInsert<"pets">;

async function createPet(
  pet: Omit<PetInsert, "id" | "created_at" | "updated_at">,
): Promise<Pet | null> {
  const { data, error } = await supabase
    .from("pets")
    .insert(pet)
    .select()
    .single();

  if (error) {
    console.error("Error al crear mascota:", error.message);
    return null;
  }

  return data;
}
```

## ❤️ Favoritos

### Agregar a Favoritos

```typescript
async function addToFavorites(petId: string) {
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    console.error("Usuario no autenticado");
    return null;
  }

  const { data, error } = await supabase
    .from("favorites")
    .insert({
      user_id: user.id,
      pet_id: petId,
    })
    .select()
    .single();

  if (error) {
    console.error("Error al agregar favorito:", error.message);
    return null;
  }

  return data;
}
```

### Obtener Favoritos del Usuario

```typescript
async function getUserFavorites() {
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) return [];

  const { data, error } = await supabase
    .from("favorites")
    .select(
      `
      *,
      pets (*)
    `,
    )
    .eq("user_id", user.id);

  if (error) {
    console.error("Error al obtener favoritos:", error.message);
    return [];
  }

  return data;
}
```

### Eliminar de Favoritos

```typescript
async function removeFromFavorites(favoriteId: string) {
  const { error } = await supabase
    .from("favorites")
    .delete()
    .eq("id", favoriteId);

  if (error) {
    console.error("Error al eliminar favorito:", error.message);
    return false;
  }

  return true;
}
```

## 📝 Solicitudes de Adopción

### Crear Solicitud de Adopción

```typescript
async function createAdoptionRequest(petId: string) {
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    console.error("Usuario no autenticado");
    return null;
  }

  const { data, error } = await supabase
    .from("adoption_requests")
    .insert({
      user_id: user.id,
      pet_id: petId,
      estado: "Pendiente",
    })
    .select()
    .single();

  if (error) {
    console.error("Error al crear solicitud:", error.message);
    return null;
  }

  return data;
}
```

### Obtener Solicitudes del Usuario

```typescript
async function getUserAdoptionRequests() {
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) return [];

  const { data, error } = await supabase
    .from("adoption_requests")
    .select(
      `
      *,
      pets (*)
    `,
    )
    .eq("user_id", user.id)
    .order("fecha", { ascending: false });

  if (error) {
    console.error("Error al obtener solicitudes:", error.message);
    return [];
  }

  return data;
}
```

### Actualizar Estado de Solicitud (Solo Admin)

```typescript
async function updateAdoptionRequestStatus(
  requestId: string,
  estado: "Pendiente" | "Aprobada" | "Rechazada",
) {
  const { data, error } = await supabase
    .from("adoption_requests")
    .update({ estado })
    .eq("id", requestId)
    .select()
    .single();

  if (error) {
    console.error("Error al actualizar solicitud:", error.message);
    return null;
  }

  return data;
}
```

## 🔄 Suscripciones en Tiempo Real

### Escuchar Cambios en Mascotas

```typescript
import { useEffect } from "react";

function usePetsSubscription(onPetChange: (pet: Pet) => void) {
  useEffect(() => {
    const channel = supabase
      .channel("pets-changes")
      .on(
        "postgres_changes",
        {
          event: "*",
          schema: "public",
          table: "pets",
        },
        (payload) => {
          console.log("Cambio en mascotas:", payload);
          if (payload.new) {
            onPetChange(payload.new as Pet);
          }
        },
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [onPetChange]);
}
```

## 👤 Perfil de Usuario

### Obtener Perfil del Usuario

```typescript
async function getUserProfile(userId: string) {
  const { data, error } = await supabase
    .from("users")
    .select("*")
    .eq("id", userId)
    .single();

  if (error) {
    console.error("Error al obtener perfil:", error.message);
    return null;
  }

  return data;
}
```

### Actualizar Perfil del Usuario

```typescript
import type { TablesUpdate } from "@/types/database.types";

type UserUpdate = TablesUpdate<"users">;

async function updateUserProfile(updates: UserUpdate) {
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    console.error("Usuario no autenticado");
    return null;
  }

  const { data, error } = await supabase
    .from("users")
    .update(updates)
    .eq("id", user.id)
    .select()
    .single();

  if (error) {
    console.error("Error al actualizar perfil:", error.message);
    return null;
  }

  return data;
}
```
