# kristal-shop
================================================================================
Kristal Dünyası - COMPLETE SOURCE CODE - الكود الكامل
================================================================================
Project Name: kristal-dunyasi
Version: 1.0.0
Date: November 2024
================================================================================


================================================================================
1. DATABASE SCHEMA (drizzle/schema.ts)
================================================================================
import { int, mysqlEnum, mysqlTable, text, timestamp, varchar } from "drizzle-orm/mysql-core";

/**
 * Core user table backing auth flow.
 * Extend this file with additional tables as your product grows.
 * Columns use camelCase to match both database fields and generated types.
 */
export const users = mysqlTable("users", {
  /**
   * Surrogate primary key. Auto-incremented numeric value managed by the database.
   * Use this for relations between tables.
   */
  id: int("id").autoincrement().primaryKey(),
  /** Manus OAuth identifier (openId) returned from the OAuth callback. Unique per user. */
  openId: varchar("openId", { length: 64 }).notNull().unique(),
  name: text("name"),
  email: varchar("email", { length: 320 }),
  loginMethod: varchar("loginMethod", { length: 64 }),
  role: mysqlEnum("role", ["user", "admin"]).default("user").notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
  lastSignedIn: timestamp("lastSignedIn").defaultNow().notNull(),
});

export type User = typeof users.$inferSelect;
export type InsertUser = typeof users.$inferInsert;

/**
 * Products table - stores all products listed for sale
 */
export const products = mysqlTable("products", {
  id: int("id").autoincrement().primaryKey(),
  /** Seller's user ID */
  sellerId: int("sellerId").notNull(),
  /** Product name */
  name: varchar("name", { length: 255 }).notNull(),
  /** Product description */
  description: text("description"),
  /** Product price in cents to avoid floating point issues */
  price: int("price").notNull(),
  /** Product category */
  category: varchar("category", { length: 100 }),
  /** Product image URL */
  imageUrl: text("imageUrl"),
  /** Product status - available or sold */
  status: mysqlEnum("status", ["available", "sold", "pending"]).default("available").notNull(),
  /** Stock quantity */
  quantity: int("quantity").default(1).notNull(),
  /** Timestamps */
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
});

export type Product = typeof products.$inferSelect;
export type InsertProduct = typeof products.$inferInsert;

/**
 * Orders table - stores purchase transactions
 */
export const orders = mysqlTable("orders", {
  id: int("id").autoincrement().primaryKey(),
  /** Buyer's user ID */
  buyerId: int("buyerId").notNull(),
  /** Seller's user ID */
  sellerId: int("sellerId").notNull(),
  /** Product ID being purchased */
  productId: int("productId").notNull(),
  /** Quantity purchased */
  quantity: int("quantity").default(1).notNull(),
  /** Price at time of purchase (in cents) */
  price: int("price").notNull(),
  /** Order status */
  status: mysqlEnum("status", ["pending", "completed", "cancelled", "refunded"]).default("pending").notNull(),
  /** Timestamps */
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
});

export type Order = typeof orders.$inferSelect;
export type InsertOrder = typeof orders.$inferInsert;

================================================================================
2. DATABASE QUERIES (server/db.ts)
================================================================================
import { eq } from "drizzle-orm";
import { drizzle } from "drizzle-orm/mysql2";
import { InsertUser, users, products, orders, InsertProduct, InsertOrder } from "../drizzle/schema";
import { ENV } from './_core/env';

let _db: ReturnType<typeof drizzle> | null = null;

// Lazily create the drizzle instance so local tooling can run without a DB.
export async function getDb() {
  if (!_db && process.env.DATABASE_URL) {
    try {
      _db = drizzle(process.env.DATABASE_URL);
    } catch (error) {
      console.warn("[Database] Failed to connect:", error);
      _db = null;
    }
  }
  return _db;
}

export async function upsertUser(user: InsertUser): Promise<void> {
  if (!user.openId) {
    throw new Error("User openId is required for upsert");
  }

  const db = await getDb();
  if (!db) {
    console.warn("[Database] Cannot upsert user: database not available");
    return;
  }

  try {
    const values: InsertUser = {
      openId: user.openId,
    };
    const updateSet: Record<string, unknown> = {};

    const textFields = ["name", "email", "loginMethod"] as const;
    type TextField = (typeof textFields)[number];

    const assignNullable = (field: TextField) => {
      const value = user[field];
      if (value === undefined) return;
      const normalized = value ?? null;
      values[field] = normalized;
      updateSet[field] = normalized;
    };

    textFields.forEach(assignNullable);

    if (user.lastSignedIn !== undefined) {
      values.lastSignedIn = user.lastSignedIn;
      updateSet.lastSignedIn = user.lastSignedIn;
    }
    if (user.role !== undefined) {
      values.role = user.role;
      updateSet.role = user.role;
    } else if (user.openId === ENV.ownerOpenId) {
      values.role = 'admin';
      updateSet.role = 'admin';
    }

    if (!values.lastSignedIn) {
      values.lastSignedIn = new Date();
    }

    if (Object.keys(updateSet).length === 0) {
      updateSet.lastSignedIn = new Date();
    }

    await db.insert(users).values(values).onDuplicateKeyUpdate({
      set: updateSet,
    });
  } catch (error) {
    console.error("[Database] Failed to upsert user:", error);
    throw error;
  }
}

export async function getUserByOpenId(openId: string) {
  const db = await getDb();
  if (!db) {
    console.warn("[Database] Cannot get user: database not available");
    return undefined;
  }

  const result = await db.select().from(users).where(eq(users.openId, openId)).limit(1);

  return result.length > 0 ? result[0] : undefined;
}

// Product queries
export async function createProduct(product: InsertProduct) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  const result = await db.insert(products).values(product);
  return result;
}

export async function getProductById(productId: number) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  const result = await db.select().from(products).where(eq(products.id, productId)).limit(1);
  return result.length > 0 ? result[0] : null;
}

export async function getAllProducts(limit = 50, offset = 0) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.select().from(products).where(eq(products.status, 'available')).limit(limit).offset(offset);
}

export async function getUserProducts(userId: number) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.select().from(products).where(eq(products.sellerId, userId));
}

export async function updateProduct(productId: number, updates: Partial<InsertProduct>) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.update(products).set(updates).where(eq(products.id, productId));
}

// Order queries
export async function createOrder(order: InsertOrder) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.insert(orders).values(order);
}

export async function getUserPurchases(userId: number) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.select().from(orders).where(eq(orders.buyerId, userId));
}

export async function getUserSales(userId: number) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.select().from(orders).where(eq(orders.sellerId, userId));
}

export async function getOrderById(orderId: number) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  const result = await db.select().from(orders).where(eq(orders.id, orderId)).limit(1);
  return result.length > 0 ? result[0] : null;
}

export async function updateOrder(orderId: number, updates: Partial<InsertOrder>) {
  const db = await getDb();
  if (!db) throw new Error("Database not available");
  
  return await db.update(orders).set(updates).where(eq(orders.id, orderId));
}


================================================================================
3. TRPC ROUTERS (server/routers.ts)
================================================================================
import { COOKIE_NAME } from "@shared/const";
import { getSessionCookieOptions } from "./_core/cookies";
import { systemRouter } from "./_core/systemRouter";
import { publicProcedure, router, protectedProcedure } from "./_core/trpc";
import { z } from "zod";
import { createProduct, getAllProducts, getProductById, getUserProducts, updateProduct, createOrder, getUserPurchases, getUserSales, getOrderById } from "./db";

export const appRouter = router({
  system: systemRouter,
  
  auth: router({
    me: publicProcedure.query(opts => opts.ctx.user),
    logout: publicProcedure.mutation(({ ctx }) => {
      const cookieOptions = getSessionCookieOptions(ctx.req);
      ctx.res.clearCookie(COOKIE_NAME, { ...cookieOptions, maxAge: -1 });
      return {
        success: true,
      } as const;
    }),
  }),

  // Product routes
  products: router({
    list: publicProcedure
      .input(z.object({ limit: z.number().default(50), offset: z.number().default(0) }).optional())
      .query(async ({ input }) => {
        const { limit = 50, offset = 0 } = input || {};
        return await getAllProducts(limit, offset);
      }),
    
    getById: publicProcedure
      .input(z.number())
      .query(async ({ input }) => {
        return await getProductById(input);
      }),
    
    create: protectedProcedure
      .input(z.object({
        name: z.string().min(1),
        description: z.string().optional(),
        price: z.number().int().positive(),
        category: z.string().optional(),
        imageUrl: z.string().optional(),
        quantity: z.number().int().default(1),
      }))
      .mutation(async ({ input, ctx }) => {
        return await createProduct({
          sellerId: ctx.user.id,
          ...input,
        });
      }),
    
    getUserProducts: protectedProcedure
      .query(async ({ ctx }) => {
        return await getUserProducts(ctx.user.id);
      }),
    
    update: protectedProcedure
      .input(z.object({
        id: z.number(),
        updates: z.object({
          name: z.string().optional(),
          description: z.string().optional(),
          price: z.number().int().optional(),
          category: z.string().optional(),
          imageUrl: z.string().optional(),
          quantity: z.number().int().optional(),
          status: z.enum(["available", "sold", "pending"]).optional(),
        }),
      }))
      .mutation(async ({ input, ctx }) => {
        const product = await getProductById(input.id);
        if (!product || product.sellerId !== ctx.user.id) {
          throw new Error("Not authorized");
        }
        return await updateProduct(input.id, input.updates);
      }),
  }),

  // Order routes
  orders: router({
    create: protectedProcedure
      .input(z.object({
        productId: z.number(),
        quantity: z.number().int().positive().default(1),
      }))
      .mutation(async ({ input, ctx }) => {
        const product = await getProductById(input.productId);
        if (!product) {
          throw new Error("Product not found");
        }
        if (product.quantity < input.quantity) {
          throw new Error("Insufficient quantity");
        }
        
        const order = await createOrder({
          buyerId: ctx.user.id,
          sellerId: product.sellerId,
          productId: input.productId,
          quantity: input.quantity,
          price: product.price * input.quantity,
          status: "completed",
        });
        
        // Update product quantity
        await updateProduct(input.productId, {
          quantity: product.quantity - input.quantity,
          status: product.quantity - input.quantity === 0 ? "sold" : "available",
        });
        
        return order;
      }),
    
    getPurchases: protectedProcedure
      .query(async ({ ctx }) => {
        return await getUserPurchases(ctx.user.id);
      }),
    
    getSales: protectedProcedure
      .query(async ({ ctx }) => {
        return await getUserSales(ctx.user.id);
      }),
    
    getById: protectedProcedure
      .input(z.number())
      .query(async ({ input, ctx }) => {
        const order = await getOrderById(input);
        if (!order || (order.buyerId !== ctx.user.id && order.sellerId !== ctx.user.id)) {
          throw new Error("Not authorized");
        }
        return order;
      }),
  }),
});

export type AppRouter = typeof appRouter;


================================================================================
4. PRODUCT CARD COMPONENT (client/src/components/ProductCard.tsx)
================================================================================
import { Product } from "@shared/types";
import { Button } from "@/components/ui/button";
import { Card } from "@/components/ui/card";
import { ShoppingCart, Sparkles } from "lucide-react";
import { useState } from "react";

interface ProductCardProps {
  product: Product;
  onBuy?: (productId: number) => void;
  isLoading?: boolean;
}

export default function ProductCard({ product, onBuy, isLoading }: ProductCardProps) {
  const [isHovered, setIsHovered] = useState(false);
  
  const priceInDollars = (product.price / 100).toFixed(2);

  return (
    <Card 
      className="relative overflow-hidden group transition-all duration-300 hover:shadow-2xl hover:scale-105 cursor-pointer h-full flex flex-col"
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
    >
      {/* Magical gradient background */}
      <div className="absolute inset-0 bg-gradient-to-br from-indigo-100 via-transparent to-blue-100 opacity-0 group-hover:opacity-100 transition-opacity duration-300" />
      
      {/* Sparkle effect */}
      {isHovered && (
        <div className="absolute inset-0 pointer-events-none">
          <div className="absolute top-2 right-2 animate-pulse">
            <Sparkles className="w-4 h-4 text-indigo-400" />
          </div>
          <div className="absolute bottom-3 left-3 animate-pulse" style={{ animationDelay: "0.2s" }}>
            <Sparkles className="w-3 h-3 text-blue-300" />
          </div>
        </div>
      )}

      {/* Product Image Container */}
      <div className="relative w-full h-48 bg-gradient-to-b from-indigo-50 to-blue-50 overflow-hidden flex items-center justify-center">
        {product.imageUrl ? (
          <img 
            src={product.imageUrl} 
            alt={product.name}
            className="w-full h-full object-cover group-hover:scale-110 transition-transform duration-300"
          />
        ) : (
          <div className="flex flex-col items-center justify-center text-indigo-300">
            <div className="w-16 h-16 rounded-full border-2 border-indigo-200 flex items-center justify-center">
              <Sparkles className="w-8 h-8" />
            </div>
            <p className="text-xs text-indigo-400 mt-2">صورة غير متاحة</p>
          </div>
        )}
        
        {/* Status Badge */}
        {product.status === "sold" && (
          <div className="absolute inset-0 bg-black/40 flex items-center justify-center">
            <span className="text-white font-bold text-lg">مباع</span>
          </div>
        )}
        
        {/* Quantity Badge */}
        {product.quantity > 0 && (
          <div className="absolute top-2 left-2 bg-indigo-600 text-white px-3 py-1 rounded-full text-xs font-semibold">
            {product.quantity} متاح
          </div>
        )}
      </div>

      {/* Content */}
      <div className="relative z-10 flex-1 p-4 flex flex-col">
        {/* Product Name */}
        <h3 className="font-bold text-lg text-foreground line-clamp-2 group-hover:text-indigo-600 transition-colors">
          {product.name}
        </h3>

        {/* Category */}
        {product.category && (
          <p className="text-xs text-muted-foreground mt-1 uppercase tracking-wide">
            {product.category}
          </p>
        )}

        {/* Description */}
        {product.description && (
          <p className="text-sm text-muted-foreground mt-2 line-clamp-2 flex-1">
            {product.description}
          </p>
        )}

        {/* Price Section */}
        <div className="mt-4 pt-4 border-t border-border">
          <div className="flex items-baseline justify-between">
            <span className="text-xs text-muted-foreground">السعر</span>
            <span className="text-2xl font-bold text-indigo-600">
              ${priceInDollars}
            </span>
          </div>
        </div>

        {/* Buy Button */}
        <Button
          onClick={() => onBuy?.(product.id)}
          disabled={isLoading || product.status === "sold" || product.quantity === 0}
          className="w-full mt-4 bg-gradient-to-r from-indigo-600 to-blue-600 hover:from-indigo-700 hover:to-blue-700 text-white font-semibold transition-all duration-300 group-hover:shadow-lg"
        >
          <ShoppingCart className="w-4 h-4 mr-2" />
          {isLoading ? "جاري..." : product.status === "sold" ? "مباع" : "شراء الآن"}
        </Button>
      </div>

      {/* Magical border effect on hover */}
      <div className="absolute inset-0 rounded-lg border-2 border-transparent group-hover:border-indigo-300 transition-colors duration-300 pointer-events-none" 
           style={{
             backgroundImage: isHovered 
               ? "linear-gradient(45deg, transparent 30%, rgba(99, 102, 241, 0.1) 50%, transparent 70%)"
               : "none",
             backgroundSize: "200% 200%",
             animation: isHovered ? "shimmer 3s infinite" : "none"
           }}
      />
    </Card>
  );
}

// Add keyframe animation for shimmer effect
const style = document.createElement("style");
style.textContent = `
  @keyframes shimmer {
    0% { background-position: -200% -200%; }
    100% { background-position: 200% 200%; }
  }
`;
document.head.appendChild(style);


================================================================================
5. HOME PAGE (client/src/pages/Home.tsx)
================================================================================
import { useAuth } from "@/_core/hooks/useAuth";
import { Button } from "@/components/ui/button";
import { trpc } from "@/lib/trpc";
import { APP_LOGO, APP_TITLE, getLoginUrl } from "@/const";
import { Loader2, Sparkles, ShoppingBag } from "lucide-react";
import ProductCard from "@/components/ProductCard";
import { useState } from "react";
import { useLocation } from "wouter";

export default function Home() {
  const { user, loading: authLoading, isAuthenticated } = useAuth();
  const [, navigate] = useLocation();
  const [selectedCategory, setSelectedCategory] = useState<string | null>(null);

  // Fetch products
  const { data: products, isLoading: productsLoading } = trpc.products.list.useQuery({
    limit: 50,
    offset: 0,
  });

  // Create order mutation
  const createOrderMutation = trpc.orders.create.useMutation({
    onSuccess: () => {
      // Refresh products after purchase
      
