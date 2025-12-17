# 🌱 Sürdürülebilirlik Modülü
# Oluşturulma Tarihi: 2024-12-17


---

## 📋 İÇİNDEKİLER

1. **Kurulum Gereksinimleri** (package.json dependencies)
2. **Veritabanı Şeması** (shared/schema.ts - Sustainability bölümü)
3. **Backend API Routes** (server/routes.ts - Sustainability endpoints)
4. **Storage Interface** (server/storage.ts - Sustainability methods)
5. **Emisyon Parametreleri** (57 parametre - seed data)
6. **Frontend Sayfaları**:
   - sustainability-dashboard.tsx
   - sustainability-data-entry.tsx
   - sustainability-category-detail.tsx (Referans implementasyon)
   - sustainability-parameters.tsx
   - sustainability-analysis.tsx
   - sustainability-analytics.tsx
   - sustainability-reports.tsx
   - sustainability-water-footprint.tsx
7. **Yardımcı Kütüphaneler** (lib/numberFormat.ts)
8. **Routing Yapılandırması** (App.tsx routing)

---

## 1. KURULUM GEREKSİNİMLERİ

### Package.json Dependencies

```json
{
  "dependencies": {
    "@hookform/resolvers": "^3.9.0",
    "@neondatabase/serverless": "^0.9.5",
    "@radix-ui/react-accordion": "^1.2.2",
    "@radix-ui/react-alert-dialog": "^1.1.6",
    "@radix-ui/react-checkbox": "^1.1.4",
    "@radix-ui/react-dialog": "^1.1.6",
    "@radix-ui/react-dropdown-menu": "^2.1.6",
    "@radix-ui/react-label": "^2.1.2",
    "@radix-ui/react-popover": "^1.1.6",
    "@radix-ui/react-select": "^2.1.6",
    "@radix-ui/react-separator": "^1.1.2",
    "@radix-ui/react-slot": "^1.1.2",
    "@radix-ui/react-switch": "^1.1.3",
    "@radix-ui/react-tabs": "^1.1.3",
    "@radix-ui/react-toast": "^1.2.6",
    "@radix-ui/react-tooltip": "^1.1.8",
    "@tanstack/react-query": "^5.60.5",
    "bcrypt": "^5.1.1",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "date-fns": "^3.6.0",
    "drizzle-kit": "^0.24.0",
    "drizzle-orm": "^0.33.0",
    "drizzle-zod": "^0.5.1",
    "express": "^4.21.0",
    "express-session": "^1.18.0",
    "jsonwebtoken": "^9.0.2",
    "lucide-react": "^0.453.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-hook-form": "^7.53.2",
    "recharts": "^2.13.3",
    "tailwind-merge": "^2.5.4",
    "tailwindcss": "^3.4.14",
    "tailwindcss-animate": "^1.0.7",
    "vite": "^5.4.6",
    "wouter": "^3.3.5",
    "zod": "^3.23.8"
  }
}
```

---

## 2. VERİTABANI ŞEMASI

### shared/schema.ts - Sustainability Tabloları

```typescript
import { pgTable, varchar, text, boolean, timestamp, integer } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";
import { sql } from "drizzle-orm";

// Sustainability - Emission Parameters (Factors for calculations)
export const emissionParameters = pgTable("emission_parameters", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  category: text("category").notNull(), // e.g., "Kategori 1.1", "Kategori 1.4"
  title: text("title").notNull(), // e.g., "Sabit Yanma", "Hareketli Yanma"
  emissionSource: text("emission_source").notNull(), // e.g., "İstme Amaçlı Doğalgaz - TR"
  emissionFactor: text("emission_factor").notNull(), // Numeric value stored as text to preserve precision
  unit: text("unit").notNull(), // e.g., "kgCO2e/Sm³", "kgCO2e/litre"
  source: text("source").notNull(), // e.g., "IPCC Vol.2 Ch.2 Tablo 2.3", "Defra 2024"
  
  // Metadata
  isActive: boolean("is_active").default(true),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

// Schema for inserting emission parameters
export const insertEmissionParameterSchema = createInsertSchema(emissionParameters).omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

export type InsertEmissionParameter = z.infer<typeof insertEmissionParameterSchema>;
export type EmissionParameter = typeof emissionParameters.$inferSelect;

// Sustainability - Emission Data
export const sustainabilityEmissions = pgTable("sustainability_emissions", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  locationId: varchar("location_id").notNull(), // Hospital/facility
  userId: varchar("user_id").notNull(), // Data entry user
  
  // Scope and Category
  scope: text("scope").notNull(), // "scope1", "scope2", "scope3"
  categoryId: text("category_id").notNull(), // e.g., "1.1", "1.2", "2.1", "3.1", etc.
  categoryName: text("category_name").notNull(), // e.g., "Kategori 1.1", "Kategori 2.1"
  
  // Emission Source
  sourceName: text("source_name").notNull(), // e.g., "Jeneratör - Benzin", "Elektrik Tüketimi"
  sourceUnit: text("source_unit").notNull(), // e.g., "litre", "kWh", "kg", "m³"
  
  // Period and Amount
  month: text("month"), // YYYY-MM format (for non-electricity sources)
  period: text("period"), // For electricity: "YYYY-DD" format (year-period number)
  amount: text("amount").notNull(), // Store as text to preserve precision
  
  // Electricity-specific fields
  electricitySupplierLocation: text("electricity_supplier_location"), // "Elektrik Tüketimi - TR", etc.
  
  // Natural gas-specific fields
  naturalGasSupplierLocation: text("natural_gas_supplier_location"), // "Isıtma Amaçlı Doğalgaz - TR", etc.
  
  // Common fields for electricity and natural gas
  startDate: timestamp("start_date"), // For electricity/gas: başlangıç okuma tarihi
  endDate: timestamp("end_date"), // For electricity/gas: bitiş okuma tarihi
  
  // File upload
  uploadedFile: text("uploaded_file"), // URL to uploaded PDF/PNG/JPEG
  
  // Additional Information
  notes: text("notes"), // Optional notes
  
  // Calculated Emissions
  co2Equivalent: text("co2_equivalent"), // Calculated CO2 equivalent in kg or tons
  emissionFactor: text("emission_factor"), // Emission factor used for calculation
  
  // Metadata
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

// Schema for inserting sustainability emissions
export const insertSustainabilityEmissionSchema = createInsertSchema(sustainabilityEmissions).omit({
  id: true,
  createdAt: true,
  updatedAt: true,
}).extend({
  startDate: z.string().optional(),
  endDate: z.string().optional(),
  co2Equivalent: z.string().optional(),
  emissionFactor: z.string().optional(),
});

export type InsertSustainabilityEmission = z.infer<typeof insertSustainabilityEmissionSchema>;
export type SustainabilityEmission = typeof sustainabilityEmissions.$inferSelect;
```

---

## 3. BACKEND API ROUTES

### server/routes.ts - Sustainability Endpoints

```typescript
import { Express, Request, Response } from "express";
import { storage } from "./storage";
import { insertSustainabilityEmissionSchema, insertEmissionParameterSchema } from "@shared/schema";

// Middleware for authentication (implement based on your auth system)
const authenticateToken = (req: Request, res: Response, next: Function) => {
  // Your authentication logic here
  next();
};

const requireSustainAdmin = (req: Request, res: Response, next: Function) => {
  const user = (req as any).user;
  if (!['central_admin', 'sustain_admin'].includes(user?.role)) {
    return res.status(403).json({ message: 'Bu işlem için yetkiniz bulunmamaktadır' });
  }
  next();
};

export function registerSustainabilityRoutes(app: Express) {
  // ==================== SUSTAINABILITY EMISSIONS ROUTES ====================
  
  // Get all hospitals for sustain_admin to select when entering data
  app.get('/api/sustainability/hospitals', authenticateToken, requireSustainAdmin, async (req: Request, res: Response) => {
    try {
      const allLocations = await storage.getAllLocations();
      res.json(allLocations);
    } catch (error: any) {
      console.error('Get sustainability hospitals error:', error);
      res.status(500).json({ message: 'Hastane listesi alınırken hata oluştu: ' + error.message });
    }
  });
  
  // Get all emissions (with optional location filter)
  app.get('/api/sustainability/emissions', authenticateToken, async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const locationId = req.query.locationId as string | undefined;
      
      const filterLocationId = ['central_admin', 'sustain_admin'].includes(user.role) ? locationId : user.locationId;
      const emissions = await storage.getAllSustainabilityEmissions(filterLocationId);
      
      res.json(emissions);
    } catch (error: any) {
      console.error('Get emissions error:', error);
      res.status(500).json({ message: 'Emisyon verileri alınırken hata oluştu: ' + error.message });
    }
  });

  // Get dashboard emissions summary
  app.get('/api/sustainability/emissions/dashboard', authenticateToken, async (req: Request, res: Response) => {
    try {
      const { year, locationId } = req.query;
      
      if (!year) {
        return res.status(400).json({ message: 'year parametresi gereklidir' });
      }
      
      const dashboardData = await storage.getDashboardEmissions(
        year as string,
        locationId as string | undefined
      );
      
      res.json(dashboardData);
    } catch (error: any) {
      console.error('Dashboard emissions fetch error:', error);
      res.status(500).json({ message: 'Dashboard verisi alınırken hata oluştu: ' + error.message });
    }
  });

  // Get emissions by source (for a specific emission source)
  app.get('/api/sustainability/emissions/source', authenticateToken, async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { locationId, scope, categoryId, sourceName } = req.query;
      
      if (!scope || !categoryId || !sourceName) {
        return res.status(400).json({ message: 'scope, categoryId ve sourceName parametreleri gereklidir' });
      }
      
      const filterLocationId = ['central_admin', 'sustain_admin'].includes(user.role) 
        ? (locationId as string | undefined)
        : user.locationId;
      
      const emissions = await storage.getEmissionsBySource(
        filterLocationId,
        scope as string,
        categoryId as string,
        sourceName as string
      );
      
      res.json(emissions);
    } catch (error: any) {
      console.error('Get emissions by source error:', error);
      res.status(500).json({ message: 'Emisyon verileri alınırken hata oluştu: ' + error.message });
    }
  });

  // Get single emission
  app.get('/api/sustainability/emissions/:id', authenticateToken, async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const emission = await storage.getSustainabilityEmission(req.params.id);
      
      if (!emission) {
        return res.status(404).json({ message: 'Emisyon verisi bulunamadı' });
      }
      
      if (!['central_admin', 'sustain_admin'].includes(user.role) && emission.locationId !== user.locationId) {
        return res.status(403).json({ message: 'Bu veriye erişim yetkiniz bulunmamaktadır' });
      }
      
      res.json(emission);
    } catch (error: any) {
      console.error('Get emission error:', error);
      res.status(500).json({ message: 'Emisyon verisi alınırken hata oluştu: ' + error.message });
    }
  });

  // Create emission entry
  app.post('/api/sustainability/emissions', authenticateToken, async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const validatedData = insertSustainabilityEmissionSchema.parse(req.body);
      
      if (!['central_admin', 'sustain_admin'].includes(user.role) && validatedData.locationId !== user.locationId) {
        return res.status(403).json({ message: 'Sadece kendi lokasyonunuz için veri girebilirsiniz' });
      }
      
      const emissionData: any = {
        ...validatedData,
        userId: user.id
      };
      
      if (validatedData.startDate) {
        emissionData.startDate = new Date(validatedData.startDate);
      }
      if (validatedData.endDate) {
        emissionData.endDate = new Date(validatedData.endDate);
      }
      
      // Calculate CO2 equivalent if emissionFactor is provided
      if (validatedData.emissionFactor && validatedData.amount) {
        const amount = parseFloat(validatedData.amount);
        const factor = parseFloat(validatedData.emissionFactor);
        
        if (isNaN(amount) || isNaN(factor)) {
          return res.status(400).json({ message: 'Geçersiz tutar veya emisyon faktörü' });
        }
        
        if (factor === 0) {
          return res.status(400).json({ message: 'Emisyon faktörü 0 olamaz. Lütfen geçerli bir tedarikçi seçin.' });
        }
        
        const co2InKg = amount * factor;
        emissionData.co2Equivalent = co2InKg.toString();
      }
      
      const newEmission = await storage.createSustainabilityEmission(emissionData);
      
      res.status(201).json(newEmission);
    } catch (error: any) {
      console.error('Create emission error:', error);
      if (error.name === 'ZodError') {
        return res.status(400).json({ message: 'Geçersiz veri formatı', errors: error.errors });
      }
      res.status(500).json({ message: 'Emisyon verisi oluşturulurken hata oluştu: ' + error.message });
    }
  });

  // Update emission entry
  app.put('/api/sustainability/emissions/:id', authenticateToken, async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const existingEmission = await storage.getSustainabilityEmission(req.params.id);
      
      if (!existingEmission) {
        return res.status(404).json({ message: 'Emisyon verisi bulunamadı' });
      }
      
      if (!['central_admin', 'sustain_admin'].includes(user.role) && existingEmission.locationId !== user.locationId) {
        return res.status(403).json({ message: 'Bu veriyi güncelleme yetkiniz bulunmamaktadır' });
      }
      
      const validatedData = insertSustainabilityEmissionSchema.partial().parse(req.body);
      
      const emissionData: any = { ...validatedData };
      
      if (validatedData.startDate) {
        emissionData.startDate = new Date(validatedData.startDate);
      }
      if (validatedData.endDate) {
        emissionData.endDate = new Date(validatedData.endDate);
      }
      
      // Recalculate CO2 equivalent if amount or emissionFactor changed
      const shouldRecalculate = validatedData.amount || validatedData.emissionFactor;
      
      if (shouldRecalculate && (existingEmission.amount || existingEmission.emissionFactor)) {
        const amount = validatedData.amount 
          ? parseFloat(validatedData.amount) 
          : parseFloat(existingEmission.amount);
        const factor = validatedData.emissionFactor 
          ? parseFloat(validatedData.emissionFactor) 
          : parseFloat(existingEmission.emissionFactor || "0");
        
        if (!isNaN(amount) && !isNaN(factor) && factor !== 0) {
          const co2InKg = amount * factor;
          emissionData.co2Equivalent = co2InKg.toString();
        }
      }
      
      const updatedEmission = await storage.updateSustainabilityEmission(req.params.id, emissionData);
      
      res.json(updatedEmission);
    } catch (error: any) {
      console.error('Update emission error:', error);
      if (error.name === 'ZodError') {
        return res.status(400).json({ message: 'Geçersiz veri formatı', errors: error.errors });
      }
      res.status(500).json({ message: 'Emisyon verisi güncellenirken hata oluştu: ' + error.message });
    }
  });

  // Delete emission entry
  app.delete('/api/sustainability/emissions/:id', authenticateToken, async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const existingEmission = await storage.getSustainabilityEmission(req.params.id);
      
      if (!existingEmission) {
        return res.status(404).json({ message: 'Emisyon verisi bulunamadı' });
      }
      
      if (!['central_admin', 'sustain_admin'].includes(user.role) && existingEmission.locationId !== user.locationId) {
        return res.status(403).json({ message: 'Bu veriyi silme yetkiniz bulunmamaktadır' });
      }
      
      const deleted = await storage.deleteSustainabilityEmission(req.params.id);
      
      if (deleted) {
        res.json({ message: 'Emisyon verisi başarıyla silindi' });
      } else {
        res.status(500).json({ message: 'Emisyon verisi silinirken hata oluştu' });
      }
    } catch (error: any) {
      console.error('Delete emission error:', error);
      res.status(500).json({ message: 'Emisyon verisi silinirken hata oluştu: ' + error.message });
    }
  });

  // ========== Emission Parameters Endpoints ==========
  
  // Get all emission parameters
  app.get('/api/emission-parameters', authenticateToken, async (req: Request, res: Response) => {
    try {
      const parameters = await storage.getEmissionParameters();
      res.json(parameters);
    } catch (error: any) {
      console.error('Get emission parameters error:', error);
      res.status(500).json({ message: 'Emisyon parametreleri yüklenirken hata oluştu: ' + error.message });
    }
  });

  // Create new emission parameter
  app.post('/api/emission-parameters', authenticateToken, requireSustainAdmin, async (req: Request, res: Response) => {
    try {
      const validatedData = insertEmissionParameterSchema.parse(req.body);
      const newParameter = await storage.createEmissionParameter(validatedData);
      res.status(201).json(newParameter);
    } catch (error: any) {
      console.error('Create emission parameter error:', error);
      if (error.name === 'ZodError') {
        return res.status(400).json({ message: 'Geçersiz veri formatı', errors: error.errors });
      }
      res.status(500).json({ message: 'Emisyon parametresi oluşturulurken hata oluştu: ' + error.message });
    }
  });

  // Update emission parameter
  app.patch('/api/emission-parameters/:id', authenticateToken, requireSustainAdmin, async (req: Request, res: Response) => {
    try {
      const validatedData = insertEmissionParameterSchema.partial().parse(req.body);
      const updatedParameter = await storage.updateEmissionParameter(req.params.id, validatedData);
      
      if (!updatedParameter) {
        return res.status(404).json({ message: 'Emisyon parametresi bulunamadı' });
      }
      
      res.json(updatedParameter);
    } catch (error: any) {
      console.error('Update emission parameter error:', error);
      if (error.name === 'ZodError') {
        return res.status(400).json({ message: 'Geçersiz veri formatı', errors: error.errors });
      }
      res.status(500).json({ message: 'Emisyon parametresi güncellenirken hata oluştu: ' + error.message });
    }
  });

  // Delete emission parameter
  app.delete('/api/emission-parameters/:id', authenticateToken, requireSustainAdmin, async (req: Request, res: Response) => {
    try {
      const deleted = await storage.deleteEmissionParameter(req.params.id);
      
      if (deleted) {
        res.json({ message: 'Emisyon parametresi başarıyla silindi' });
      } else {
        res.status(404).json({ message: 'Emisyon parametresi bulunamadı' });
      }
    } catch (error: any) {
      console.error('Delete emission parameter error:', error);
      res.status(500).json({ message: 'Emisyon parametresi silinirken hata oluştu: ' + error.message });
    }
  });
}
```

---

## 4. STORAGE INTERFACE

### server/storage.ts - Sustainability Methods

```typescript
import { db } from "./db";
import { eq, and, desc, sql } from "drizzle-orm";
import { sustainabilityEmissions, emissionParameters } from "@shared/schema";
import type { SustainabilityEmission, InsertSustainabilityEmission, EmissionParameter, InsertEmissionParameter } from "@shared/schema";

export interface SourceEmission {
  sourceName: string;
  categoryId: string;
  categoryName: string;
  total: number;
  percentage: number;
}

export interface ScopeEmissions {
  total: number;
  sources: SourceEmission[];
}

export interface DashboardEmissionsData {
  year: string;
  locationId: string | null;
  totalEmissions: number;
  scope1: ScopeEmissions;
  scope2: ScopeEmissions;
  scope3: ScopeEmissions;
}

export interface IStorageSustainability {
  // Sustainability Emissions operations
  getAllSustainabilityEmissions(locationId?: string): Promise<SustainabilityEmission[]>;
  getSustainabilityEmission(id: string): Promise<SustainabilityEmission | undefined>;
  getEmissionsBySource(locationId: string | undefined, scope: string, categoryId: string, sourceName: string): Promise<SustainabilityEmission[]>;
  getDashboardEmissions(year: string, locationId?: string): Promise<DashboardEmissionsData>;
  createSustainabilityEmission(emission: InsertSustainabilityEmission): Promise<SustainabilityEmission>;
  updateSustainabilityEmission(id: string, emission: Partial<InsertSustainabilityEmission>): Promise<SustainabilityEmission>;
  deleteSustainabilityEmission(id: string): Promise<boolean>;

  // Emission Parameters operations
  getEmissionParameters(): Promise<EmissionParameter[]>;
  getEmissionParameter(id: string): Promise<EmissionParameter | undefined>;
  createEmissionParameter(parameter: InsertEmissionParameter): Promise<EmissionParameter>;
  updateEmissionParameter(id: string, parameter: Partial<InsertEmissionParameter>): Promise<EmissionParameter | null>;
  deleteEmissionParameter(id: string): Promise<boolean>;
}

export class SustainabilityStorage implements IStorageSustainability {
  // Sustainability Emissions operations
  async getAllSustainabilityEmissions(locationId?: string): Promise<SustainabilityEmission[]> {
    if (locationId) {
      return await db.select().from(sustainabilityEmissions)
        .where(eq(sustainabilityEmissions.locationId, locationId))
        .orderBy(desc(sustainabilityEmissions.month), desc(sustainabilityEmissions.createdAt));
    }
    return await db.select().from(sustainabilityEmissions)
      .orderBy(desc(sustainabilityEmissions.month), desc(sustainabilityEmissions.createdAt));
  }

  async getSustainabilityEmission(id: string): Promise<SustainabilityEmission | undefined> {
    const [emission] = await db.select().from(sustainabilityEmissions)
      .where(eq(sustainabilityEmissions.id, id));
    return emission;
  }

  async getEmissionsBySource(locationId: string | undefined, scope: string, categoryId: string, sourceName: string): Promise<SustainabilityEmission[]> {
    const conditions = [
      eq(sustainabilityEmissions.scope, scope),
      eq(sustainabilityEmissions.categoryId, categoryId),
      eq(sustainabilityEmissions.sourceName, sourceName)
    ];
    
    if (locationId) {
      conditions.push(eq(sustainabilityEmissions.locationId, locationId));
    }
    
    return await db.select().from(sustainabilityEmissions)
      .where(and(...conditions))
      .orderBy(desc(sustainabilityEmissions.month));
  }

  async getDashboardEmissions(year: string, locationId?: string): Promise<DashboardEmissionsData> {
    const conditions = [
      sql`${sustainabilityEmissions.period} LIKE ${year + '%'}`
    ];
    
    if (locationId) {
      conditions.push(eq(sustainabilityEmissions.locationId, locationId));
    }
    
    const emissions = await db.select().from(sustainabilityEmissions)
      .where(and(...conditions));
    
    const scope1Emissions: { [key: string]: SourceEmission } = {};
    const scope2Emissions: { [key: string]: SourceEmission } = {};
    const scope3Emissions: { [key: string]: SourceEmission } = {};
    
    let scope1Total = 0;
    let scope2Total = 0;
    let scope3Total = 0;
    
    emissions.forEach(emission => {
      const co2InTons = parseFloat(emission.co2Equivalent || "0") / 1000;
      const key = `${emission.sourceName}|${emission.categoryId}|${emission.categoryName}`;
      
      if (emission.scope === "scope1") {
        if (!scope1Emissions[key]) {
          scope1Emissions[key] = {
            sourceName: emission.sourceName || "",
            categoryId: emission.categoryId || "",
            categoryName: emission.categoryName || "",
            total: 0,
            percentage: 0
          };
        }
        scope1Emissions[key].total += co2InTons;
        scope1Total += co2InTons;
      } else if (emission.scope === "scope2") {
        if (!scope2Emissions[key]) {
          scope2Emissions[key] = {
            sourceName: emission.sourceName || "",
            categoryId: emission.categoryId || "",
            categoryName: emission.categoryName || "",
            total: 0,
            percentage: 0
          };
        }
        scope2Emissions[key].total += co2InTons;
        scope2Total += co2InTons;
      } else if (emission.scope === "scope3") {
        if (!scope3Emissions[key]) {
          scope3Emissions[key] = {
            sourceName: emission.sourceName || "",
            categoryId: emission.categoryId || "",
            categoryName: emission.categoryName || "",
            total: 0,
            percentage: 0
          };
        }
        scope3Emissions[key].total += co2InTons;
        scope3Total += co2InTons;
      }
    });
    
    const scope1Sources = Object.values(scope1Emissions).map(source => ({
      ...source,
      percentage: scope1Total > 0 ? (source.total / scope1Total) * 100 : 0
    }));
    
    const scope2Sources = Object.values(scope2Emissions).map(source => ({
      ...source,
      percentage: scope2Total > 0 ? (source.total / scope2Total) * 100 : 0
    }));
    
    const scope3Sources = Object.values(scope3Emissions).map(source => ({
      ...source,
      percentage: scope3Total > 0 ? (source.total / scope3Total) * 100 : 0
    }));
    
    return {
      year,
      locationId: locationId || null,
      totalEmissions: scope1Total + scope2Total + scope3Total,
      scope1: {
        total: scope1Total,
        sources: scope1Sources
      },
      scope2: {
        total: scope2Total,
        sources: scope2Sources
      },
      scope3: {
        total: scope3Total,
        sources: scope3Sources
      }
    };
  }

  async createSustainabilityEmission(emission: InsertSustainabilityEmission): Promise<SustainabilityEmission> {
    const [newEmission] = await db
      .insert(sustainabilityEmissions)
      .values(emission)
      .returning();
    return newEmission;
  }

  async updateSustainabilityEmission(id: string, emission: Partial<InsertSustainabilityEmission>): Promise<SustainabilityEmission> {
    const [updatedEmission] = await db
      .update(sustainabilityEmissions)
      .set({ 
        ...emission, 
        updatedAt: new Date()
      })
      .where(eq(sustainabilityEmissions.id, id))
      .returning();
    return updatedEmission;
  }

  async deleteSustainabilityEmission(id: string): Promise<boolean> {
    const result = await db.delete(sustainabilityEmissions).where(eq(sustainabilityEmissions.id, id));
    return (result.rowCount || 0) > 0;
  }

  // Emission Parameters operations
  async getEmissionParameters(): Promise<EmissionParameter[]> {
    return await db.select().from(emissionParameters)
      .where(eq(emissionParameters.isActive, true))
      .orderBy(emissionParameters.category, emissionParameters.title);
  }

  async getEmissionParameter(id: string): Promise<EmissionParameter | undefined> {
    const [parameter] = await db.select().from(emissionParameters)
      .where(eq(emissionParameters.id, id));
    return parameter;
  }

  async createEmissionParameter(parameter: InsertEmissionParameter): Promise<EmissionParameter> {
    const [newParameter] = await db
      .insert(emissionParameters)
      .values(parameter)
      .returning();
    return newParameter;
  }

  async updateEmissionParameter(id: string, parameter: Partial<InsertEmissionParameter>): Promise<EmissionParameter | null> {
    const [updatedParameter] = await db
      .update(emissionParameters)
      .set({ 
        ...parameter, 
        updatedAt: new Date()
      })
      .where(eq(emissionParameters.id, id))
      .returning();
    return updatedParameter || null;
  }

  async deleteEmissionParameter(id: string): Promise<boolean> {
    const result = await db.delete(emissionParameters).where(eq(emissionParameters.id, id));
    return (result.rowCount || 0) > 0;
  }
}

export const sustainabilityStorage = new SustainabilityStorage();
```

---

## 5. EMİSYON PARAMETRELERİ (57 Parametre)

### server/seed-data/emission-parameters.ts

```typescript
// Emission Parameters for Sustainability Module
// Total: 57 parameters from IPCC and Defra sources

export const emissionParametersSeed = [
  // =============== SCOPE 1: DIRECT EMISSIONS ===============
  
  // Kategori 1.1 - Sabit Yanma
  { category: "Kategori 1.1", title: "Sabit Yanma", emissionSource: "Jeneratör - Benzin", emissionFactor: "2.324", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.2 Tablo 2.3" },
  { category: "Kategori 1.1", title: "Sabit Yanma", emissionSource: "Jeneratör - Dizel", emissionFactor: "2.614", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.2 Tablo 2.3" },
  { category: "Kategori 1.1", title: "Sabit Yanma", emissionSource: "Fuel Oil", emissionFactor: "2.966", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.2 Tablo 2.3" },
  { category: "Kategori 1.1", title: "Sabit Yanma", emissionSource: "Isıtma Amaçlı Doğalgaz - TR", emissionFactor: "1.943", unit: "kgCO2e/Sm³", source: "IPCC Vol.2 Ch.2 Tablo 2.3" },
  { category: "Kategori 1.1", title: "Sabit Yanma", emissionSource: "Isıtma Amaçlı Doğalgaz - EU/Diğer", emissionFactor: "2.021", unit: "kgCO2e/Sm³", source: "Defra 2024" },
  
  // Kategori 1.2 - Hareketli Yanma
  { category: "Kategori 1.2", title: "Hareketli Yanma", emissionSource: "Operasyonel Araçlar - Dizel", emissionFactor: "2.614", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.3 Tablo 3.2" },
  { category: "Kategori 1.2", title: "Hareketli Yanma", emissionSource: "Şirket Araçları - Benzin", emissionFactor: "2.324", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.3 Tablo 3.2" },
  { category: "Kategori 1.2", title: "Hareketli Yanma", emissionSource: "Şirket Araçları - Dizel", emissionFactor: "2.614", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.3 Tablo 3.2" },
  { category: "Kategori 1.2", title: "Hareketli Yanma", emissionSource: "Şirket Araçları - LPG", emissionFactor: "1.506", unit: "kgCO2e/litre", source: "IPCC Vol.2 Ch.3 Tablo 3.2" },
  
  // Kategori 1.4 - Kaçak Emisyonlar (Anestezi Gazları)
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "Isoflurane", emissionFactor: "510", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "Sevorane/Sevoflurane", emissionFactor: "130", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "Suprane/Desflurane", emissionFactor: "2540", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "Sojourn Inhalasyon için Uçucu Çöz.", emissionFactor: "130", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "Nitrous Oxide (N2O)", emissionFactor: "298", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  
  // Kategori 1.4 - Kaçak Emisyonlar (Soğutucu Gazlar)
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "R-410A", emissionFactor: "2088", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "R-32", emissionFactor: "675", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "R-134a", emissionFactor: "1430", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "R-407C", emissionFactor: "1774", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "R-404A", emissionFactor: "3922", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "R-22", emissionFactor: "1810", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  
  // Kategori 1.4 - Yangın Söndürme Sistemleri
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "CO2 Yangın Tüpleri", emissionFactor: "1", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "FM-200 (HFC-227ea)", emissionFactor: "3220", unit: "kgCO2e/kg", source: "IPCC AR5 GWP-100" },
  { category: "Kategori 1.4", title: "Kaçak Emisyonlar", emissionSource: "Novec 1230", emissionFactor: "1", unit: "kgCO2e/kg", source: "3M Technical Data" },
  
  // =============== SCOPE 2: INDIRECT ENERGY EMISSIONS ===============
  
  // Kategori 2.1 - Satın Alınan Elektrik
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - TR", emissionFactor: "0.440", unit: "kgCO2e/kWh", source: "EPDK 2024 - Türkiye Şebeke Ortalaması" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Macaristan", emissionFactor: "0.233", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Romanya", emissionFactor: "0.293", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Kazakistan", emissionFactor: "0.636", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Kuzey Kıbrıs", emissionFactor: "0.440", unit: "kgCO2e/kWh", source: "EPDK 2024 (TR proxy)" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Azerbaycan", emissionFactor: "0.480", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Özbekistan", emissionFactor: "0.550", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Almanya", emissionFactor: "0.350", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Bulgaristan", emissionFactor: "0.410", unit: "kgCO2e/kWh", source: "IEA 2024" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Irak", emissionFactor: "0.700", unit: "kgCO2e/kWh", source: "IEA 2024 Estimate" },
  { category: "Kategori 2.1", title: "Satın Alınan Elektrik", emissionSource: "Elektrik Tüketimi - Makedonya", emissionFactor: "0.565", unit: "kgCO2e/kWh", source: "IEA 2024" },
  
  // =============== SCOPE 3: OTHER INDIRECT EMISSIONS ===============
  
  // Kategori 3.1 - Satın Alınan Mal ve Hizmetler
  { category: "Kategori 3.1", title: "Satın Alınan Mal ve Hizmetler", emissionSource: "Mal Taşımacılığı - Karayolu", emissionFactor: "0.107", unit: "kgCO2e/ton-km", source: "Defra 2024" },
  { category: "Kategori 3.1", title: "Satın Alınan Mal ve Hizmetler", emissionSource: "Mal Taşımacılığı - Denizyolu", emissionFactor: "0.016", unit: "kgCO2e/ton-km", source: "Defra 2024" },
  { category: "Kategori 3.1", title: "Satın Alınan Mal ve Hizmetler", emissionSource: "Mal Taşımacılığı - Havayolu", emissionFactor: "0.602", unit: "kgCO2e/ton-km", source: "Defra 2024" },
  { category: "Kategori 3.1", title: "Satın Alınan Mal ve Hizmetler", emissionSource: "Mal Taşımacılığı - Demiryolu", emissionFactor: "0.028", unit: "kgCO2e/ton-km", source: "Defra 2024" },
  
  // Kategori 3.3 - Yakıt ve Enerji ile İlgili Faaliyetler
  { category: "Kategori 3.3", title: "Yakıt ve Enerji ile İlgili Faaliyetler", emissionSource: "Servis - Benzin", emissionFactor: "0.161", unit: "kgCO2e/km", source: "Defra 2024" },
  { category: "Kategori 3.3", title: "Yakıt ve Enerji ile İlgili Faaliyetler", emissionSource: "Servis - Dizel", emissionFactor: "0.168", unit: "kgCO2e/km", source: "Defra 2024" },
  { category: "Kategori 3.3", title: "Yakıt ve Enerji ile İlgili Faaliyetler", emissionSource: "Servis - Elektrik", emissionFactor: "0.053", unit: "kgCO2e/km", source: "Defra 2024" },
  
  // Kategori 3.5 - İş Seyahatleri
  { category: "Kategori 3.5", title: "İş Seyahatleri", emissionSource: "Uçuş - Kısa Mesafe (<500km)", emissionFactor: "0.255", unit: "kgCO2e/yolcu-km", source: "Defra 2024" },
  { category: "Kategori 3.5", title: "İş Seyahatleri", emissionSource: "Uçuş - Orta Mesafe (500-3700km)", emissionFactor: "0.156", unit: "kgCO2e/yolcu-km", source: "Defra 2024" },
  { category: "Kategori 3.5", title: "İş Seyahatleri", emissionSource: "Uçuş - Uzun Mesafe (>3700km)", emissionFactor: "0.150", unit: "kgCO2e/yolcu-km", source: "Defra 2024" },
  { category: "Kategori 3.5", title: "İş Seyahatleri", emissionSource: "Taksi", emissionFactor: "0.149", unit: "kgCO2e/km", source: "Defra 2024" },
  
  // Kategori 4.1 - Yukarı Akış Taşımacılık ve Dağıtım (Su)
  { category: "Kategori 4.1", title: "Yukarı Akış - Su", emissionSource: "Şebeke Suyu Kullanımı", emissionFactor: "0.149", unit: "kgCO2e/m³", source: "Defra 2024" },
  { category: "Kategori 4.1", title: "Yukarı Akış - Su", emissionSource: "Atık Su Arıtımı (WWT)", emissionFactor: "0.272", unit: "kgCO2e/m³", source: "Defra 2024" },
  
  // Kategori 4.3 - Faaliyetlerden Kaynaklanan Atıklar
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Tehlikeli Atık - Yakma", emissionFactor: "0.025", unit: "kgCO2e/kg", source: "Defra 2024" },
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Tehlikeli Atık - Bertaraf", emissionFactor: "0.013", unit: "kgCO2e/kg", source: "Defra 2024" },
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Tıbbi Atık - Yakma", emissionFactor: "0.025", unit: "kgCO2e/kg", source: "Defra 2024" },
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Evsel Atık - Düzenli Depolama", emissionFactor: "0.586", unit: "kgCO2e/kg", source: "Defra 2024" },
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Plastik - Geri Dönüşüm", emissionFactor: "0.021", unit: "kgCO2e/kg", source: "Defra 2024" },
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Kağıt - Geri Dönüşüm", emissionFactor: "0.021", unit: "kgCO2e/kg", source: "Defra 2024" },
  { category: "Kategori 4.3", title: "Atık Yönetimi", emissionSource: "Cam - Geri Dönüşüm", emissionFactor: "0.021", unit: "kgCO2e/kg", source: "Defra 2024" },
  
  // Kategori 4.5 - Kiralanan Varlıklar (Hizmet Alımları)
  { category: "Kategori 4.5", title: "Kiralanan Varlıklar", emissionSource: "Temizlik Hizmeti", emissionFactor: "0.520", unit: "kgCO2e/m²", source: "Defra 2024 - Business Services" },
  { category: "Kategori 4.5", title: "Kiralanan Varlıklar", emissionSource: "Güvenlik Hizmeti", emissionFactor: "0.050", unit: "kgCO2e/saat", source: "Defra 2024 Estimate" },
  { category: "Kategori 4.5", title: "Kiralanan Varlıklar", emissionSource: "Yemek Hizmeti", emissionFactor: "2.500", unit: "kgCO2e/öğün", source: "Defra 2024 - Food Services" },
];

// Seed function to insert parameters into database
export async function seedEmissionParameters(storage: any) {
  console.log('Seeding emission parameters...');
  
  for (const param of emissionParametersSeed) {
    try {
      await storage.createEmissionParameter({
        category: param.category,
        title: param.title,
        emissionSource: param.emissionSource,
        emissionFactor: param.emissionFactor,
        unit: param.unit,
        source: param.source,
        isActive: true
      });
    } catch (error: any) {
      if (!error.message?.includes('duplicate')) {
        console.error(`Error seeding parameter: ${param.emissionSource}`, error.message);
      }
    }
  }
  
  console.log(`Seeded ${emissionParametersSeed.length} emission parameters`);
}
```

---

## 6. FRONTEND SAYFALARI

### 6.1 sustainability-dashboard.tsx

(Aşağıda sustainability-dashboard.tsx dosyasının tam içeriği yer alacak)


---

### 6.1 sustainability-dashboard.tsx

```tsx
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useAuth } from "@/hooks/useAuth";
import { BarChart3, TrendingUp, Leaf, Download, Palette } from "lucide-react";
import { formatTurkishNumber } from "@/lib/numberFormat";
import { PieChart, Pie, Cell, ResponsiveContainer, Legend, Tooltip } from "recharts";
import html2canvas from "html2canvas";
import type { Location } from "@shared/schema";

interface SourceEmission {
  sourceName: string;
  categoryId: string;
  categoryName: string;
  total: number;
  percentage: number;
}

interface ScopeEmissions {
  total: number;
  sources: SourceEmission[];
}

interface DashboardData {
  year: string;
  locationId: string | null;
  totalEmissions: number;
  scope1: ScopeEmissions;
  scope2: ScopeEmissions;
  scope3: ScopeEmissions;
}

const currentYear = new Date().getFullYear();
const years = Array.from({ length: 10 }, (_, i) => currentYear - i);

// Default colors for scopes
const DEFAULT_SCOPE_COLORS = {
  scope1: '#ef4444', // Red
  scope2: '#22c55e', // Green
  scope3: '#a855f7', // Purple
};

// PNG export function
const downloadChartAsPNG = async (chartId: string, filename: string) => {
  const chartElement = document.getElementById(chartId);
  if (!chartElement) return;

  try {
    const canvas = await html2canvas(chartElement, {
      backgroundColor: '#ffffff',
      scale: 2,
    });
    
    const link = document.createElement('a');
    link.download = filename;
    link.href = canvas.toDataURL('image/png');
    link.click();
  } catch (error) {
    console.error('PNG export error:', error);
  }
};

// Default source colors (vibrant and distinct)
const DEFAULT_SOURCE_COLORS = [
  '#3b82f6', // Blue
  '#ef4444', // Red
  '#22c55e', // Green
  '#f59e0b', // Amber
  '#8b5cf6', // Violet
  '#ec4899', // Pink
  '#14b8a6', // Teal
  '#f97316', // Orange
  '#a855f7', // Purple
  '#06b6d4', // Cyan
];

interface ScopeBreakdownProps {
  scopeName: string;
  scopeKey: string;
  scopeData: ScopeEmissions;
  baseColor: string;
}

function ScopeBreakdown({ scopeName, scopeKey, scopeData, baseColor }: ScopeBreakdownProps) {
  const chartId = `scope-breakdown-${scopeKey}`;
  
  // Source colors (customizable per scope)
  const [sourceColors, setSourceColors] = useState<Record<string, string>>(() => {
    try {
      const saved = localStorage.getItem(`scope-colors-${scopeKey}`);
      if (saved) return JSON.parse(saved);
      
      // Initialize with default colors
      const colors: Record<string, string> = {};
      scopeData.sources.forEach((source, index) => {
        colors[source.categoryId] = DEFAULT_SOURCE_COLORS[index % DEFAULT_SOURCE_COLORS.length];
      });
      return colors;
    } catch {
      const colors: Record<string, string> = {};
      scopeData.sources.forEach((source, index) => {
        colors[source.categoryId] = DEFAULT_SOURCE_COLORS[index % DEFAULT_SOURCE_COLORS.length];
      });
      return colors;
    }
  });

  const updateSourceColor = (categoryId: string, color: string) => {
    const newColors = { ...sourceColors, [categoryId]: color };
    setSourceColors(newColors);
    localStorage.setItem(`scope-colors-${scopeKey}`, JSON.stringify(newColors));
  };

  // Prepare data for pie chart
  const pieData = scopeData.sources.map(source => ({
    name: source.categoryName,
    value: source.total,
    percentage: source.percentage,
    categoryId: source.categoryId,
  }));

  const hasData = scopeData.sources.length > 0 && scopeData.total > 0;

  if (!hasData) {
    return (
      <Card>
        <CardHeader>
          <CardTitle style={{ color: baseColor }}>
            {scopeName} - Detaylı Analiz
          </CardTitle>
          <CardDescription>
            Bu kapsam için veri bulunamadı
          </CardDescription>
        </CardHeader>
        <CardContent>
          <div className="h-96 flex items-center justify-center text-gray-500">
            Veri yok
          </div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card id={chartId}>
      <CardHeader>
        <div className="flex items-center justify-between">
          <div>
            <CardTitle style={{ color: baseColor }}>
              {scopeName} - Detaylı Analiz
            </CardTitle>
            <CardDescription>
              Toplam: {formatTurkishNumber(scopeData.total)} tCO2e
            </CardDescription>
          </div>
          <Button
            variant="outline"
            size="sm"
            onClick={() => downloadChartAsPNG(chartId, `${scopeKey}-detay.png`)}
            data-testid={`button-export-${scopeKey}`}
          >
            <Download className="h-4 w-4 mr-2" />
            PNG İndir
          </Button>
        </div>
      </CardHeader>
      <CardContent>
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
          {/* Left: Pie Chart */}
          <div className="space-y-4">
            <h3 className="font-semibold text-gray-700 dark:text-gray-300">
              Kaynak Dağılımı
            </h3>
            <ResponsiveContainer width="100%" height={300}>
              <PieChart>
                <Pie
                  data={pieData}
                  cx="50%"
                  cy="50%"
                  labelLine={false}
                  label={({ percentage }) => `%${percentage.toFixed(1)}`}
                  outerRadius={100}
                  fill="#8884d8"
                  dataKey="value"
                >
                  {pieData.map((entry) => (
                    <Cell 
                      key={entry.categoryId} 
                      fill={sourceColors[entry.categoryId] || DEFAULT_SOURCE_COLORS[0]} 
                    />
                  ))}
                </Pie>
                <Tooltip 
                  formatter={(value: number) => `${formatTurkishNumber(value)} tCO2e`}
                />
                <Legend />
              </PieChart>
            </ResponsiveContainer>

            {/* Color customization */}
            <div className="space-y-2">
              <h4 className="text-sm font-medium text-gray-600 dark:text-gray-400">
                Renk Özelleştirme
              </h4>
              <div className="grid grid-cols-2 gap-2">
                {scopeData.sources.map((source) => (
                  <div key={source.categoryId} className="flex items-center gap-2">
                    <Input
                      type="color"
                      value={sourceColors[source.categoryId] || DEFAULT_SOURCE_COLORS[0]}
                      onChange={(e) => updateSourceColor(source.categoryId, e.target.value)}
                      className="w-12 h-8 p-1 cursor-pointer"
                      data-testid={`input-color-${scopeKey}-${source.categoryId}`}
                    />
                    <span className="text-xs text-gray-600 dark:text-gray-400 truncate">
                      {source.categoryName}
                    </span>
                  </div>
                ))}
              </div>
            </div>
          </div>

          {/* Right: Progress Bars */}
          <div className="space-y-4">
            <h3 className="font-semibold text-gray-700 dark:text-gray-300">
              Kaynak Detayları
            </h3>
            <div className="space-y-3">
              {[...scopeData.sources]
                .sort((a, b) => b.total - a.total)
                .map((source) => (
                  <div key={source.categoryId} className="space-y-1">
                    <div className="flex justify-between items-center">
                      <span className="text-sm font-medium text-gray-700 dark:text-gray-300">
                        {source.categoryName}
                      </span>
                      <span className="text-sm text-gray-600 dark:text-gray-400">
                        {formatTurkishNumber(source.total)} tCO2e ({source.percentage.toFixed(1)}%)
                      </span>
                    </div>
                    <div 
                      className="w-full bg-gray-200 dark:bg-gray-700 rounded-full h-3"
                      data-testid={`progress-${scopeKey}-${source.categoryId}`}
                    >
                      <div
                        className="h-3 rounded-full transition-all duration-300"
                        style={{
                          width: `${source.percentage}%`,
                          backgroundColor: sourceColors[source.categoryId] || DEFAULT_SOURCE_COLORS[0],
                        }}
                      />
                    </div>
                  </div>
                ))}
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}

export default function SustainabilityDashboard() {
  const { user } = useAuth();
  const [selectedYear, setSelectedYear] = useState(currentYear.toString());
  const [selectedHospital, setSelectedHospital] = useState<string | null>(null);
  
  // Scope colors (customizable)
  const [scopeColors, setScopeColors] = useState<Record<string, string>>(() => {
    try {
      const saved = localStorage.getItem('dashboard-scope-colors');
      return saved ? JSON.parse(saved) : DEFAULT_SCOPE_COLORS;
    } catch {
      return DEFAULT_SCOPE_COLORS;
    }
  });

  const updateScopeColor = (scope: string, color: string) => {
    const newColors = { ...scopeColors, [scope]: color };
    setScopeColors(newColors);
    localStorage.setItem('dashboard-scope-colors', JSON.stringify(newColors));
  };

  // Fetch dashboard data
  const dashboardUrl = `/api/sustainability/emissions/dashboard?year=${selectedYear}${selectedHospital ? `&locationId=${selectedHospital}` : ''}`;
  const { data: dashboardData, isLoading: isDashboardLoading } = useQuery<DashboardData>({
    queryKey: [dashboardUrl],
    enabled: !!selectedYear,
  });

  // Fetch hospitals for filter (sustain_admin only)
  const { data: hospitals = [] } = useQuery<Location[]>({
    queryKey: ['/api/sustainability/hospitals'],
    enabled: user?.role === 'sustain_admin',
  });

  if (!user) {
    return <div className="p-6">Yükleniyor...</div>;
  }

  // Calculate scope percentages for global pie chart
  const totalEmissions = dashboardData?.totalEmissions || 0;
  const scope1Percentage = totalEmissions > 0 ? (dashboardData?.scope1.total || 0) / totalEmissions * 100 : 0;
  const scope2Percentage = totalEmissions > 0 ? (dashboardData?.scope2.total || 0) / totalEmissions * 100 : 0;
  const scope3Percentage = totalEmissions > 0 ? (dashboardData?.scope3.total || 0) / totalEmissions * 100 : 0;

  return (
    <div className="min-h-screen bg-gradient-to-br from-green-50 via-blue-50 to-purple-50 dark:from-gray-900 dark:via-gray-800 dark:to-gray-900 p-6">
      <div className="max-w-7xl mx-auto space-y-6">
        {/* Header */}
        <div className="flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="h-12 w-12 rounded-lg bg-gradient-to-br from-green-500 to-blue-600 flex items-center justify-center">
              <Leaf className="h-6 w-6 text-white" />
            </div>
            <div>
              <h1 className="text-3xl font-bold text-gray-900 dark:text-white">
                Sürdürülebilirlik Dashboard
              </h1>
              <p className="text-gray-600 dark:text-gray-400">
                Emisyon analizleri ve raporlama
              </p>
            </div>
          </div>
        </div>

        {/* Filters */}
        <Card>
          <CardHeader>
            <CardTitle className="flex items-center gap-2">
              <BarChart3 className="h-5 w-5" />
              Filtreler
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              {/* Year Filter */}
              <div className="space-y-2">
                <label className="text-sm font-medium">Yıl</label>
                <Select value={selectedYear} onValueChange={setSelectedYear}>
                  <SelectTrigger data-testid="select-year">
                    <SelectValue placeholder="Yıl seçin" />
                  </SelectTrigger>
                  <SelectContent>
                    {years.map((year) => (
                      <SelectItem key={year} value={year.toString()}>
                        {year}
                      </SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              </div>

              {/* Hospital Filter (sustain_admin only) */}
              <div className="space-y-2">
                <label className="text-sm font-medium">Hastane</label>
                <Select 
                  value={selectedHospital || "all"} 
                  onValueChange={(value) => setSelectedHospital(value === "all" ? null : value)}
                >
                  <SelectTrigger data-testid="select-hospital">
                    <SelectValue placeholder="Hastane seçin" />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="all">Tüm Hastaneler</SelectItem>
                    {hospitals.map((hospital) => (
                      <SelectItem key={hospital.id} value={hospital.id}>
                        {hospital.shortName || hospital.name}
                      </SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              </div>
            </div>
          </CardContent>
        </Card>

        {/* Loading State */}
        {isDashboardLoading && (
          <div className="flex items-center justify-center py-12">
            <div className="animate-spin rounded-full h-12 w-12 border-4 border-primary border-t-transparent"></div>
          </div>
        )}

        {/* Dashboard Content */}
        {!isDashboardLoading && dashboardData && (
          <div className="space-y-6">
            {/* Top Summary Cards */}
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
              {/* Total Emissions */}
              <Card className="bg-gradient-to-br from-blue-500 to-blue-600 text-white">
                <CardHeader className="pb-3">
                  <CardDescription className="text-blue-100">Toplam Emisyon</CardDescription>
                  <CardTitle className="text-3xl font-bold">
                    {formatTurkishNumber(totalEmissions, 2)} tCO₂e
                  </CardTitle>
                </CardHeader>
                <CardContent>
                  <p className="text-sm text-blue-100">{selectedYear} yılı toplam</p>
                </CardContent>
              </Card>

              {/* Scope 1 */}
              <Card className="bg-gradient-to-br from-red-500 to-red-600 text-white">
                <CardHeader className="pb-3">
                  <CardDescription className="text-red-100">Kapsam 1</CardDescription>
                  <CardTitle className="text-3xl font-bold">
                    {formatTurkishNumber(dashboardData.scope1.total, 2)} tCO₂e
                  </CardTitle>
                </CardHeader>
                <CardContent>
                  <p className="text-sm text-red-100">
                    {formatTurkishNumber(scope1Percentage, 1)}% toplam emisyon
                  </p>
                </CardContent>
              </Card>

              {/* Scope 2 */}
              <Card className="bg-gradient-to-br from-green-500 to-green-600 text-white">
                <CardHeader className="pb-3">
                  <CardDescription className="text-green-100">Kapsam 2</CardDescription>
                  <CardTitle className="text-3xl font-bold">
                    {formatTurkishNumber(dashboardData.scope2.total, 2)} tCO₂e
                  </CardTitle>
                </CardHeader>
                <CardContent>
                  <p className="text-sm text-green-100">
                    {formatTurkishNumber(scope2Percentage, 1)}% toplam emisyon
                  </p>
                </CardContent>
              </Card>

              {/* Scope 3 */}
              <Card className="bg-gradient-to-br from-purple-500 to-purple-600 text-white">
                <CardHeader className="pb-3">
                  <CardDescription className="text-purple-100">Kapsam 3</CardDescription>
                  <CardTitle className="text-3xl font-bold">
                    {formatTurkishNumber(dashboardData.scope3.total, 2)} tCO₂e
                  </CardTitle>
                </CardHeader>
                <CardContent>
                  <p className="text-sm text-purple-100">
                    {formatTurkishNumber(scope3Percentage, 1)}% toplam emisyon
                  </p>
                </CardContent>
              </Card>
            </div>

            {/* Global Scope Pie Chart */}
            <Card id="global-scope-pie-chart">
              <CardHeader>
                <div className="flex items-center justify-between">
                  <div>
                    <CardTitle className="flex items-center gap-2">
                      <TrendingUp className="h-5 w-5" />
                      Kapsamlara Göre Dağılım
                    </CardTitle>
                    <CardDescription>
                      Tüm kapsamların emisyon dağılımı
                    </CardDescription>
                  </div>
                  <div className="flex items-center gap-2">
                    <Button
                      variant="outline"
                      size="sm"
                      onClick={() => downloadChartAsPNG('global-scope-pie-chart', `kapsamlar-dagilim-${selectedYear}.png`)}
                      className="flex items-center gap-2"
                      data-testid="button-export-global-pie"
                    >
                      <Download className="h-4 w-4" />
                      PNG İndir
                    </Button>
                  </div>
                </div>
              </CardHeader>
              <CardContent>
                <div className="space-y-6">
                  {/* Color Customization */}
                  <div className="flex flex-wrap gap-4 p-4 bg-gray-50 dark:bg-gray-800 rounded-lg">
                    <div className="flex items-center gap-2">
                      <Palette className="h-4 w-4 text-gray-500" />
                      <span className="text-sm font-medium">Renk Özelleştir:</span>
                    </div>
                    <div className="flex items-center gap-2">
                      <label className="text-sm">Kapsam 1:</label>
                      <Input
                        type="color"
                        value={scopeColors.scope1}
                        onChange={(e) => updateScopeColor('scope1', e.target.value)}
                        className="w-12 h-8 cursor-pointer"
                        data-testid="color-scope1"
                      />
                    </div>
                    <div className="flex items-center gap-2">
                      <label className="text-sm">Kapsam 2:</label>
                      <Input
                        type="color"
                        value={scopeColors.scope2}
                        onChange={(e) => updateScopeColor('scope2', e.target.value)}
                        className="w-12 h-8 cursor-pointer"
                        data-testid="color-scope2"
                      />
                    </div>
                    <div className="flex items-center gap-2">
                      <label className="text-sm">Kapsam 3:</label>
                      <Input
                        type="color"
                        value={scopeColors.scope3}
                        onChange={(e) => updateScopeColor('scope3', e.target.value)}
                        className="w-12 h-8 cursor-pointer"
                        data-testid="color-scope3"
                      />
                    </div>
                  </div>

                  {/* Pie Chart */}
                  {totalEmissions > 0 ? (
                    <ResponsiveContainer width="100%" height={400}>
                      <PieChart>
                        <Pie
                          data={[
                            { name: 'Kapsam 1', value: dashboardData.scope1.total, percentage: scope1Percentage },
                            { name: 'Kapsam 2', value: dashboardData.scope2.total, percentage: scope2Percentage },
                            { name: 'Kapsam 3', value: dashboardData.scope3.total, percentage: scope3Percentage },
                          ]}
                          cx="50%"
                          cy="50%"
                          labelLine={false}
                          label={(entry) => `${entry.name}: ${formatTurkishNumber(entry.percentage, 1)}%`}
                          outerRadius={120}
                          fill="#8884d8"
                          dataKey="value"
                        >
                          <Cell fill={scopeColors.scope1} />
                          <Cell fill={scopeColors.scope2} />
                          <Cell fill={scopeColors.scope3} />
                        </Pie>
                        <Tooltip 
                          formatter={(value: number) => formatTurkishNumber(value, 2) + ' tCO₂e'}
                          contentStyle={{ backgroundColor: 'white', border: '1px solid #ccc' }}
                        />
                        <Legend />
                      </PieChart>
                    </ResponsiveContainer>
                  ) : (
                    <div className="h-96 flex items-center justify-center text-gray-500">
                      Seçilen filtreler için veri bulunamadı
                    </div>
                  )}
                </div>
              </CardContent>
            </Card>

            {/* Scope 1 Breakdown */}
            <ScopeBreakdown
              scopeName="Kapsam 1"
              scopeKey="scope1"
              scopeData={dashboardData.scope1}
              baseColor={scopeColors.scope1}
            />

            {/* Scope 2 Breakdown */}
            <ScopeBreakdown
              scopeName="Kapsam 2"
              scopeKey="scope2"
              scopeData={dashboardData.scope2}
              baseColor={scopeColors.scope2}
            />

            {/* Scope 3 Breakdown */}
            <ScopeBreakdown
              scopeName="Kapsam 3"
              scopeKey="scope3"
              scopeData={dashboardData.scope3}
              baseColor={scopeColors.scope3}
            />
          </div>
        )}

        {/* Empty State */}
        {!isDashboardLoading && !dashboardData && (
          <Card>
            <CardContent className="py-12 text-center">
              <p className="text-gray-500">Seçilen filtreler için veri bulunamadı</p>
            </CardContent>
          </Card>
        )}
      </div>
    </div>
  );
}
```

---

### 6.2 sustainability-data-entry.tsx

```tsx
import { useState } from "react";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Badge } from "@/components/ui/badge";
import { useLocation } from "wouter";
import { TrendingUp, Factory, Zap, Globe, ChevronRight } from "lucide-react";

interface EmissionSource {
  id: string;
  name: string;
  categoryId: string;
  categoryName: string;
  unit: string;
  color: string;
}

const scope1Sources: EmissionSource[] = [
  { id: "gen-benzin", name: "Jeneratör - Benzin", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "litre", color: "blue" },
  { id: "gen-dizel", name: "Jeneratör - Dizel", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "litre", color: "blue" },
  { id: "fuel-oil", name: "Fuel Oil", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "litre", color: "blue" },
  { id: "dogalgaz", name: "Doğalgaz (Isıtma Amaçlı)", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "m³", color: "blue" },
  { id: "op-dizel", name: "Operasyonel Araçlar - Dizel", categoryId: "1.2", categoryName: "Kategori 1.2 - Hareketli Yakma Kaynakları", unit: "litre", color: "indigo" },
  { id: "sirket-benzin", name: "Şirket Araçları - Benzin", categoryId: "1.2", categoryName: "Kategori 1.2 - Hareketli Yakma Kaynakları", unit: "litre", color: "indigo" },
  { id: "sirket-dizel", name: "Şirket Araçları - Dizel", categoryId: "1.2", categoryName: "Kategori 1.2 - Hareketli Yakma Kaynakları", unit: "litre", color: "indigo" },
  { id: "isoflurane", name: "Isoflurane", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg", color: "purple" },
  { id: "sevorane", name: "Sevorane/Sevoflurane", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg", color: "purple" },
  { id: "sogutucu", name: "Soğutucu Gazlar", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg", color: "purple" },
  { id: "suprane", name: "Suprane/Desflurane", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg", color: "purple" },
  { id: "yangin", name: "Yangın Tüpleri", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg", color: "purple" },
  { id: "sojourn", name: "Sojourn Inhalasyon için Ucucu Çoz.", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg", color: "purple" },
];

const scope2Sources: EmissionSource[] = [
  { id: "elektrik", name: "Elektrik Tüketimi", categoryId: "2.1", categoryName: "Kategori 2.1 - Satın Alınan Elektrik", unit: "kWh", color: "amber" },
];

const scope3Sources: EmissionSource[] = [
  { id: "mal", name: "Mal", categoryId: "3.1", categoryName: "Kategori 3.1 - Satın Alınan Mal ve Hizmetler (Taşımacılık/Dağıtım - Kuruluşa gelen)", unit: "ton-km", color: "emerald" },
  { id: "servisler", name: "Servisler", categoryId: "3.3", categoryName: "Kategori 3.3 - Yakıt ve Enerji ile İlgili Faaliyetler", unit: "km", color: "teal" },
  { id: "ucuslar", name: "Uçuşlar", categoryId: "3.5", categoryName: "Kategori 3.5 - İş Seyahatleri", unit: "km", color: "cyan" },
  { id: "su", name: "Şebeke Suyu Kullanımı", categoryId: "4.1", categoryName: "Kategori 4.1 - Yukarı Akış Taşımacılık ve Dağıtım", unit: "m³", color: "sky" },
  { id: "wwt", name: "WWT (Atık Su Arıtımı)", categoryId: "4.1", categoryName: "Kategori 4.1 - Yukarı Akış Taşımacılık ve Dağıtım", unit: "m³", color: "sky" },
  { id: "atik", name: "Karışık Tehlikeli Atık - Yakma", categoryId: "4.3", categoryName: "Kategori 4.3 - Faaliyetlerden Kaynaklanan Atıklar", unit: "kg", color: "orange" },
  { id: "temizlik", name: "Temizlik Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "ay", color: "pink" },
  { id: "yemek", name: "Yemek Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "öğün", color: "pink" },
  { id: "cevre-olcum", name: "Çevre Ölçüm Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "ölçüm", color: "pink" },
  { id: "danismanlik", name: "Danışmanlık Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "saat", color: "pink" },
  { id: "guvenlik", name: "Özel Güvenlik Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "saat", color: "pink" },
];

const colorClasses: Record<string, { border: string; bg: string }> = {
  blue: { border: "border-l-blue-500", bg: "bg-blue-50 dark:bg-blue-900" },
  indigo: { border: "border-l-indigo-500", bg: "bg-indigo-50 dark:bg-indigo-900" },
  purple: { border: "border-l-purple-500", bg: "bg-purple-50 dark:bg-purple-900" },
  amber: { border: "border-l-amber-500", bg: "bg-amber-50 dark:bg-amber-900" },
  emerald: { border: "border-l-emerald-500", bg: "bg-emerald-50 dark:bg-emerald-900" },
  teal: { border: "border-l-teal-500", bg: "bg-teal-50 dark:bg-teal-900" },
  cyan: { border: "border-l-cyan-500", bg: "bg-cyan-50 dark:bg-cyan-900" },
  sky: { border: "border-l-sky-500", bg: "bg-sky-50 dark:bg-sky-900" },
  orange: { border: "border-l-orange-500", bg: "bg-orange-50 dark:bg-orange-900" },
  pink: { border: "border-l-pink-500", bg: "bg-pink-50 dark:bg-pink-900" },
};

export default function SustainabilityDataEntry() {
  const [activeTab, setActiveTab] = useState("scope1");
  const [, setLocation] = useLocation();

  const handleSourceClick = (scope: string, source: EmissionSource) => {
    setLocation(`/sustainability/data-entry/${scope}/${source.categoryId}/${source.id}`);
  };

  const SourceCard = ({ source, scope }: { source: EmissionSource; scope: string }) => {
    const colors = colorClasses[source.color] || colorClasses.blue;
    
    return (
      <Card 
        className={`hover:shadow-lg transition-shadow cursor-pointer border-l-4 ${colors.border} ${colors.bg}`}
        onClick={() => handleSourceClick(scope, source)}
        data-testid={`card-source-${source.id}`}
      >
        <CardHeader>
          <div className="flex items-center justify-between">
            <div className="flex-1">
              <CardTitle className="text-lg flex items-center gap-2">
                {source.name}
                <Badge variant="outline" className="ml-2 text-xs">{source.unit}</Badge>
              </CardTitle>
              <CardDescription className="mt-2">
                {source.categoryName}
              </CardDescription>
            </div>
            <ChevronRight className="h-5 w-5 text-gray-400 dark:text-gray-500" />
          </div>
        </CardHeader>
      </Card>
    );
  };

  return (
    <div className="container mx-auto py-6 px-4">
      <div className="mb-6">
        <div className="flex items-center justify-between mb-2">
          <div className="flex items-center gap-3">
            <TrendingUp className="h-8 w-8 text-primary" />
            <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100">Sürdürülebilirlik Yönetimi</h1>
          </div>
          <Button 
            onClick={() => setLocation('/sustainability/analysis')}
            data-testid="button-analysis"
            className="flex items-center gap-2"
          >
            <TrendingUp className="h-4 w-4" />
            Analiz & Grafikler
          </Button>
        </div>
        <p className="text-gray-600 dark:text-gray-400">Sera gazı emisyon verilerinizi kapsam bazında girin ve takip edin</p>
      </div>

      <Tabs value={activeTab} onValueChange={setActiveTab} className="space-y-6">
        <TabsList className="grid grid-cols-3 w-full max-w-2xl bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-800">
          <TabsTrigger 
            value="scope1" 
            data-testid="tab-scope1"
            className="flex items-center gap-2 data-[state=active]:bg-primary data-[state=active]:text-white"
          >
            <Factory className="h-4 w-4" />
            Kapsam 1
          </TabsTrigger>
          <TabsTrigger 
            value="scope2" 
            data-testid="tab-scope2"
            className="flex items-center gap-2 data-[state=active]:bg-primary data-[state=active]:text-white"
          >
            <Zap className="h-4 w-4" />
            Kapsam 2
          </TabsTrigger>
          <TabsTrigger 
            value="scope3" 
            data-testid="tab-scope3"
            className="flex items-center gap-2 data-[state=active]:bg-primary data-[state=active]:text-white"
          >
            <Globe className="h-4 w-4" />
            Kapsam 3
          </TabsTrigger>
        </TabsList>

        <TabsContent value="scope1" className="space-y-4">
          <div className="mb-4">
            <h2 className="text-xl font-semibold text-gray-900 dark:text-gray-100 mb-2">Kapsam 1 - Doğrudan Emisyonlar</h2>
            <p className="text-sm text-gray-600 dark:text-gray-400">
              Kuruluşun sahip olduğu veya kontrol ettiği kaynaklardan kaynaklanan doğrudan sera gazı emisyonları
            </p>
          </div>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {scope1Sources.map((source) => (
              <SourceCard key={source.id} source={source} scope="scope1" />
            ))}
          </div>
        </TabsContent>

        <TabsContent value="scope2" className="space-y-4">
          <div className="mb-4">
            <h2 className="text-xl font-semibold text-gray-900 dark:text-gray-100 mb-2">Kapsam 2 - Dolaylı Enerji Emisyonları</h2>
            <p className="text-sm text-gray-600 dark:text-gray-400">
              Satın alınan elektrik, ısıtma ve soğutmadan kaynaklanan dolaylı sera gazı emisyonları
            </p>
          </div>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {scope2Sources.map((source) => (
              <SourceCard key={source.id} source={source} scope="scope2" />
            ))}
          </div>
        </TabsContent>

        <TabsContent value="scope3" className="space-y-4">
          <div className="mb-4">
            <h2 className="text-xl font-semibold text-gray-900 dark:text-gray-100 mb-2">Kapsam 3 - Diğer Dolaylı Emisyonlar</h2>
            <p className="text-sm text-gray-600 dark:text-gray-400">
              Değer zincirinde meydana gelen diğer dolaylı sera gazı emisyonları
            </p>
          </div>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {scope3Sources.map((source) => (
              <SourceCard key={source.id} source={source} scope="scope3" />
            ))}
          </div>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

### 6.3 sustainability-category-detail.tsx (Referans İmplementasyon - Elektrik Tüketimi)

```tsx
import { useState, useEffect, useMemo } from "react";
import { useRoute, useLocation } from "wouter";
import { useQuery, useMutation } from "@tanstack/react-query";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Badge } from "@/components/ui/badge";
import { Textarea } from "@/components/ui/textarea";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Collapsible, CollapsibleContent, CollapsibleTrigger } from "@/components/ui/collapsible";
import { useToast } from "@/hooks/use-toast";
import { useAuth } from "@/hooks/useAuth";
import { ArrowLeft, Plus, Calendar, Trash2, Upload, X, BarChart3, TrendingUp, ChevronDown, Building2, Search, Edit, Download } from "lucide-react";
import { apiRequest, queryClient } from "@/lib/queryClient";
import type { SustainabilityEmission, EmissionParameter, Location } from "@shared/schema";
import { format } from "date-fns";
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer, LineChart, Line } from "recharts";
import { formatTurkishNumber, parseTurkishNumber, handleTurkishNumberKeyPress, formatAsYouType, calculateEmission } from "@/lib/numberFormat";
import html2canvas from "html2canvas";

interface SourceData {
  id: string;
  name: string;
  categoryId: string;
  categoryName: string;
  unit: string;
}

// Month labels and order for charting
const MONTH_LABELS = [
  "Ocak", "Şubat", "Mart", "Nisan", "Mayıs", "Haziran",
  "Temmuz", "Ağustos", "Eylül", "Ekim", "Kasım", "Aralık"
];

// Month colors for visual distinction
const MONTH_COLORS = [
  "#3b82f6", // Ocak - Blue
  "#10b981", // Şubat - Green
  "#8b5cf6", // Mart - Purple
  "#f59e0b", // Nisan - Amber
  "#ef4444", // Mayıs - Red
  "#06b6d4", // Haziran - Cyan
  "#ec4899", // Temmuz - Pink
  "#84cc16", // Ağustos - Lime
  "#f97316", // Eylül - Orange
  "#6366f1", // Ekim - Indigo
  "#14b8a6", // Kasım - Teal
  "#a855f7"  // Aralık - Violet
];

const allSources: Record<string, SourceData> = {
  "gen-benzin": { id: "gen-benzin", name: "Jeneratör - Benzin", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "litre" },
  "gen-dizel": { id: "gen-dizel", name: "Jeneratör - Dizel", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "litre" },
  "fuel-oil": { id: "fuel-oil", name: "Fuel Oil", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "litre" },
  "dogalgaz": { id: "dogalgaz", name: "Doğalgaz (Isıtma Amaçlı)", categoryId: "1.1", categoryName: "Kategori 1.1 - Sabit Yakma Kaynakları", unit: "m³" },
  "op-dizel": { id: "op-dizel", name: "Operasyonel Araçlar - Dizel", categoryId: "1.2", categoryName: "Kategori 1.2 - Hareketli Yakma Kaynakları", unit: "litre" },
  "sirket-benzin": { id: "sirket-benzin", name: "Şirket Araçları - Benzin", categoryId: "1.2", categoryName: "Kategori 1.2 - Hareketli Yakma Kaynakları", unit: "litre" },
  "sirket-dizel": { id: "sirket-dizel", name: "Şirket Araçları - Dizel", categoryId: "1.2", categoryName: "Kategori 1.2 - Hareketli Yakma Kaynakları", unit: "litre" },
  "isoflurane": { id: "isoflurane", name: "Isoflurane", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg" },
  "sevorane": { id: "sevorane", name: "Sevorane/Sevoflurane", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg" },
  "sogutucu": { id: "sogutucu", name: "Soğutucu Gazlar", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg" },
  "suprane": { id: "suprane", name: "Suprane/Desflurane", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg" },
  "yangin": { id: "yangin", name: "Yangın Tüpleri", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg" },
  "sojourn": { id: "sojourn", name: "Sojourn Inhalasyon için Ucucu Çoz.", categoryId: "1.4", categoryName: "Kategori 1.4 - Kaçak Emisyonlar", unit: "kg" },
  "elektrik": { id: "elektrik", name: "Elektrik Tüketimi", categoryId: "2.1", categoryName: "Kategori 2.1 - Satın Alınan Elektrik", unit: "kWh" },
  "mal": { id: "mal", name: "Mal", categoryId: "3.1", categoryName: "Kategori 3.1 - Satın Alınan Mal ve Hizmetler (Taşımacılık/Dağıtım - Kuruluşa gelen)", unit: "ton-km" },
  "servisler": { id: "servisler", name: "Servisler", categoryId: "3.3", categoryName: "Kategori 3.3 - Yakıt ve Enerji ile İlgili Faaliyetler", unit: "km" },
  "ucuslar": { id: "ucuslar", name: "Uçuşlar", categoryId: "3.5", categoryName: "Kategori 3.5 - İş Seyahatleri", unit: "km" },
  "su": { id: "su", name: "Şebeke Suyu Kullanımı", categoryId: "4.1", categoryName: "Kategori 4.1 - Yukarı Akış Taşımacılık ve Dağıtım", unit: "m³" },
  "wwt": { id: "wwt", name: "WWT (Atık Su Arıtımı)", categoryId: "4.1", categoryName: "Kategori 4.1 - Yukarı Akış Taşımacılık ve Dağıtım", unit: "m³" },
  "atik": { id: "atik", name: "Karışık Tehlikeli Atık - Yakma", categoryId: "4.3", categoryName: "Kategori 4.3 - Faaliyetlerden Kaynaklanan Atıklar", unit: "kg" },
  "temizlik": { id: "temizlik", name: "Temizlik Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "ay" },
  "yemek": { id: "yemek", name: "Yemek Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "öğün" },
  "cevre-olcum": { id: "cevre-olcum", name: "Çevre Ölçüm Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "ölçüm" },
  "danismanlik": { id: "danismanlik", name: "Danışmanlık Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "saat" },
  "guvenlik": { id: "guvenlik", name: "Özel Güvenlik Hizmeti", categoryId: "4.5", categoryName: "Kategori 4.5 - Kiralanan Varlıklar", unit: "saat" },
};

const currentYear = new Date().getFullYear();
const years = Array.from({ length: 10 }, (_, i) => currentYear - i);
const months = [
  { value: "1", label: "Ocak" },
  { value: "2", label: "Şubat" },
  { value: "3", label: "Mart" },
  { value: "4", label: "Nisan" },
  { value: "5", label: "Mayıs" },
  { value: "6", label: "Haziran" },
  { value: "7", label: "Temmuz" },
  { value: "8", label: "Ağustos" },
  { value: "9", label: "Eylül" },
  { value: "10", label: "Ekim" },
  { value: "11", label: "Kasım" },
  { value: "12", label: "Aralık" }
];

// Helper function to format period from "2024-01" to "Ocak 2024"
const formatPeriod = (period: string) => {
  if (!period || !period.includes('-')) return period;
  const [year, monthNum] = period.split('-');
  const monthName = months.find(m => m.value === parseInt(monthNum).toString())?.label || "";
  return `${monthName} ${year}`;
};

export default function SustainabilityCategoryDetail() {
  const [, params] = useRoute("/sustainability/data-entry/:scope/:categoryId/:sourceId");
  const [, setLocation] = useLocation();
  const { toast } = useToast();
  const { user } = useAuth();
  
  // Form visibility
  const [showForm, setShowForm] = useState(false);
  
  // Edit mode
  const [editingEmissionId, setEditingEmissionId] = useState<string | null>(null);
  
  // Hospital selection for sustain_admin
  const [selectedLocationId, setSelectedLocationId] = useState(user?.locationId || "");
  
  // Year and hospital selection for data display (sustain_admin)
  const [selectedYear, setSelectedYear] = useState<string | null>(null);
  const [selectedHospital, setSelectedHospital] = useState<string | null>(null);
  const [hospitalSearch, setHospitalSearch] = useState("");
  
  // Chart color customization - separate for CO2 and Consumption, per source
  const [chartColorsCO2, setChartColorsCO2] = useState<Record<string, string>>(() => {
    try {
      const storageKey = isElectricity ? 'electricity-chart-colors-co2' : 'naturalgas-chart-colors-co2';
      const saved = localStorage.getItem(storageKey);
      return saved ? JSON.parse(saved) : {};
    } catch {
      return {};
    }
  });
  
  const [chartColorsConsumption, setChartColorsConsumption] = useState<Record<string, string>>(() => {
    try {
      const storageKey = isElectricity ? 'electricity-chart-colors-consumption' : 'naturalgas-chart-colors-consumption';
      const saved = localStorage.getItem(storageKey);
      return saved ? JSON.parse(saved) : {};
    } catch {
      return {};
    }
  });
  
  const updateChartColorCO2 = (year: string, color: string) => {
    const newColors = { ...chartColorsCO2, [year]: color };
    setChartColorsCO2(newColors);
    const storageKey = isElectricity ? 'electricity-chart-colors-co2' : 'naturalgas-chart-colors-co2';
    localStorage.setItem(storageKey, JSON.stringify(newColors));
  };
  
  const updateChartColorConsumption = (year: string, color: string) => {
    const newColors = { ...chartColorsConsumption, [year]: color };
    setChartColorsConsumption(newColors);
    const storageKey = isElectricity ? 'electricity-chart-colors-consumption' : 'naturalgas-chart-colors-consumption';
    localStorage.setItem(storageKey, JSON.stringify(newColors));
  };
  
  const getYearColorCO2 = (year: string, defaultIndex: number) => {
    return chartColorsCO2[year] || `hsl(${220 + defaultIndex * 40}, 70%, 50%)`;
  };
  
  const getYearColorConsumption = (year: string, defaultIndex: number) => {
    return chartColorsConsumption[year] || `hsl(${180 + defaultIndex * 40}, 65%, 55%)`;
  };
  
  // Download chart as PNG
  const downloadChartAsPNG = async (elementId: string, fileName: string) => {
    const element = document.getElementById(elementId);
    if (!element) {
      toast({
        variant: "destructive",
        title: "Hata",
        description: "Grafik bulunamadı"
      });
      return;
    }
    
    try {
      const canvas = await html2canvas(element, {
        backgroundColor: '#ffffff',
        scale: 2,
        logging: false,
      });
      
      const link = document.createElement('a');
      link.download = `${fileName}.png`;
      link.href = canvas.toDataURL('image/png');
      link.click();
      
      toast({
        title: "Başarılı",
        description: "Grafik PNG olarak kaydedildi"
      });
    } catch (error) {
      toast({
        variant: "destructive",
        title: "Hata",
        description: "Grafik kaydedilemedi"
      });
    }
  };
  
  // Export emissions data to CSV
  const exportToCSV = (emissionsToExport: SustainabilityEmission[]) => {
    if (emissionsToExport.length === 0) {
      toast({
        variant: "destructive",
        title: "Hata",
        description: "Export edilecek veri bulunamadı"
      });
      return;
    }
    
    // CSV header - Turkish format uses semicolon (;) as separator
    const headers = [
      'Hastane',
      'Emisyon Kaynağı',
      'Dönem',
      'Başlangıç Tarihi',
      'Bitiş Tarihi',
      'Tüketim Miktarı',
      'Emisyon Değeri',
      'Notlar'
    ];
    
    // CSV rows
    const rows = emissionsToExport.map(emission => {
      const hospital = hospitals.find(h => h.id === emission.locationId);
      
      // Format period as "Ay YYYY" (e.g., "Ocak 2025")
      let period = '';
      if (emission.period) {
        const [year, monthStr] = emission.period.split('-');
        const monthIndex = parseInt(monthStr) - 1;
        period = `${MONTH_LABELS[monthIndex]} ${year}`;
      }
      
      return [
        hospital?.name || '',
        emission.electricitySupplierLocation || emission.naturalGasSupplierLocation || '',
        period,
        emission.startDate ? new Date(emission.startDate).toLocaleDateString('tr-TR') : '',
        emission.endDate ? new Date(emission.endDate).toLocaleDateString('tr-TR') : '',
        formatTurkishNumber(parseFloat(emission.amount)),
        emission.co2Equivalent ? formatTurkishNumber(parseFloat(emission.co2Equivalent)) : '',
        emission.notes || ''
      ];
    });
    
    // Combine headers and rows - using semicolon (;) separator for Turkish Excel
    const csvContent = [
      headers.join(';'),
      ...rows.map(row => row.join(';'))
    ].join('\n');
    
    // Add BOM for proper Turkish character encoding in Excel
    const BOM = '\uFEFF';
    const blob = new Blob([BOM + csvContent], { type: 'text/csv;charset=utf-8;' });
    const link = document.createElement('a');
    const url = URL.createObjectURL(blob);
    
    // Generate filename based on source type
    const fileName = isElectricity 
      ? `Elektrik-Tuketimi-Verileri-${new Date().toLocaleDateString('tr-TR').replace(/\./g, '-')}.csv`
      : `Dogalgaz-Tuketimi-Verileri-${new Date().toLocaleDateString('tr-TR').replace(/\./g, '-')}.csv`;
    
    link.setAttribute('href', url);
    link.setAttribute('download', fileName);
    link.style.visibility = 'hidden';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    
    toast({
      title: "Başarılı",
      description: `${emissionsToExport.length} kayıt CSV olarak export edildi`
    });
  };
  
  // Form fields for electricity and natural gas
  const [electricitySupplier, setElectricitySupplier] = useState("");
  const [naturalGasSupplier, setNaturalGasSupplier] = useState("");
  const [year, setYear] = useState(currentYear.toString());
  const [month, setMonth] = useState("");
  const [startDate, setStartDate] = useState("");
  const [endDate, setEndDate] = useState("");
  const [amountInteger, setAmountInteger] = useState("");
  const [amountDecimal, setAmountDecimal] = useState("");
  const [calculatedEmission, setCalculatedEmission] = useState<number>(0);
  const [notes, setNotes] = useState("");
  const [uploadedFile, setUploadedFile] = useState<File | null>(null);

  const source = params ? allSources[params.sourceId] : null;
  const isElectricity = source?.id === "elektrik";
  const isNaturalGas = source?.id === "dogalgaz";

  // Fetch all hospitals for sustain_admin
  const { data: hospitals = [] } = useQuery<Location[]>({
    queryKey: ["/api/sustainability/hospitals"],
    enabled: user?.role === "sustain_admin",
  });

  // Fetch emission parameters for electricity/natural gas suppliers
  const { data: emissionParameters = [] } = useQuery<EmissionParameter[]>({
    queryKey: ["/api/emission-parameters"],
    enabled: isElectricity || isNaturalGas,
  });

  // Filter and transform electricity suppliers from emission parameters (Category 2.1)
  const electricitySuppliers = emissionParameters
    .filter(param => param.category === "Kategori 2.1" && param.emissionSource.includes("Elektrik Tüketimi"))
    .map(param => ({
      value: param.emissionSource,
      label: param.emissionSource,
      factor: param.emissionFactor,
      unit: param.unit
    }));

  // Filter and transform natural gas suppliers from emission parameters (Category 1.1)
  const naturalGasSuppliers = emissionParameters
    .filter(param => param.category === "Kategori 1.1" && param.emissionSource.includes("Isıtma Amaçlı Doğalgaz"))
    .map(param => ({
      value: param.emissionSource,
      label: param.emissionSource,
      factor: param.emissionFactor,
      unit: param.unit
    }));

  // Set default electricity supplier when parameters are loaded (prefer TR)
  useEffect(() => {
    if (isElectricity && electricitySuppliers.length > 0 && !electricitySupplier) {
      // Try to find "Elektrik Tüketimi - TR" first
      const trSupplier = electricitySuppliers.find(s => s.value.includes("TR"));
      setElectricitySupplier(trSupplier?.value || electricitySuppliers[0].value);
    }
  }, [isElectricity, electricitySuppliers, electricitySupplier]);

  // Set default natural gas supplier when parameters are loaded (prefer TR)
  useEffect(() => {
    if (isNaturalGas && naturalGasSuppliers.length > 0 && !naturalGasSupplier) {
      // Try to find "Doğalgaz - TR" first
      const trSupplier = naturalGasSuppliers.find(s => s.value.includes("TR"));
      setNaturalGasSupplier(trSupplier?.value || naturalGasSuppliers[0].value);
    }
  }, [isNaturalGas, naturalGasSuppliers, naturalGasSupplier]);

  // Calculate emission when amount or supplier changes
  useEffect(() => {
    const currentSupplier = isElectricity ? electricitySupplier : naturalGasSupplier;
    const suppliers = isElectricity ? electricitySuppliers : naturalGasSuppliers;
    
    if ((amountInteger || amountDecimal) && currentSupplier) {
      // Combine integer and decimal parts with Turkish format
      const combinedAmount = amountDecimal 
        ? `${amountInteger || '0'},${amountDecimal}`
        : amountInteger;
      
      const selectedSupplier = suppliers.find(s => s.value === currentSupplier);
      if (selectedSupplier && combinedAmount) {
        const emission = calculateEmission(combinedAmount, selectedSupplier.factor);
        setCalculatedEmission(emission);
      }
    } else {
      setCalculatedEmission(0);
    }
  }, [amountInteger, amountDecimal, electricitySupplier, naturalGasSupplier, electricitySuppliers, naturalGasSuppliers, isElectricity]);

  // Fetch emissions for this source
  // For sustain_admin: fetch all hospitals' data, for others: only their own
  const emissionsQueryKey = user?.role === "sustain_admin"
    ? `/api/sustainability/emissions/source?scope=${params?.scope}&categoryId=${source?.categoryId}&sourceName=${encodeURIComponent(source?.name || '')}`
    : `/api/sustainability/emissions/source?locationId=${user?.locationId}&scope=${params?.scope}&categoryId=${source?.categoryId}&sourceName=${encodeURIComponent(source?.name || '')}`;
  
  const { data: emissions = [], isLoading } = useQuery<SustainabilityEmission[]>({
    queryKey: [emissionsQueryKey],
    enabled: !!(params && source),
  });

  // Create mutation
  const createMutation = useMutation({
    mutationFn: async (data: any) => {
      return await apiRequest('POST', '/api/sustainability/emissions', data);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: [`/api/sustainability/emissions/source`] });
      toast({
        title: "Başarılı",
        description: "Emisyon verisi başarıyla kaydedildi"
      });
      resetForm();
    },
    onError: (error: any) => {
      toast({
        variant: "destructive",
        title: "Hata",
        description: error.message || "Veri kaydedilirken hata oluştu"
      });
    }
  });

  // Update mutation
  const updateMutation = useMutation({
    mutationFn: async ({ id, data }: { id: string; data: any }) => {
      return await apiRequest('PUT', `/api/sustainability/emissions/${id}`, data);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: [`/api/sustainability/emissions/source`] });
      toast({
        title: "Başarılı",
        description: "Emisyon verisi başarıyla güncellendi"
      });
      resetForm();
    },
    onError: (error: any) => {
      toast({
        variant: "destructive",
        title: "Hata",
        description: error.message || "Veri güncellenirken hata oluştu"
      });
    }
  });

  // Delete mutation
  const deleteMutation = useMutation({
    mutationFn: async (id: string) => {
      return await apiRequest('DELETE', `/api/sustainability/emissions/${id}`);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: [`/api/sustainability/emissions/source`] });
      toast({
        title: "Başarılı",
        description: "Emisyon verisi başarıyla silindi"
      });
    },
    onError: (error: any) => {
      toast({
        variant: "destructive",
        title: "Hata",
        description: error.message || "Veri silinirken hata oluştu"
      });
    }
  });

  if (!params || !user || !source) {
    return <div>Yükleniyor...</div>;
  }

  const resetForm = () => {
    setShowForm(false);
    setEditingEmissionId(null);
    
    // Reset supplier based on source type
    if (isElectricity && electricitySuppliers.length > 0) {
      const trSupplier = electricitySuppliers.find(s => s.value.includes("TR"));
      setElectricitySupplier(trSupplier?.value || electricitySuppliers[0].value);
    }
    if (isNaturalGas && naturalGasSuppliers.length > 0) {
      const trSupplier = naturalGasSuppliers.find(s => s.value.includes("TR"));
      setNaturalGasSupplier(trSupplier?.value || naturalGasSuppliers[0].value);
    }
    
    setYear(currentYear.toString());
    setMonth("");
    setStartDate("");
    setEndDate("");
    setAmountInteger("");
    setAmountDecimal("");
    setCalculatedEmission(0);
    setNotes("");
    setUploadedFile(null);
  };

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      const validTypes = ['application/pdf', 'image/png', 'image/jpeg'];
      if (!validTypes.includes(file.type)) {
        toast({
          variant: "destructive",
          title: "Geçersiz Dosya",
          description: "Sadece PDF, PNG veya JPEG dosyaları yüklenebilir"
        });
        return;
      }
      setUploadedFile(file);
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (isElectricity || isNaturalGas) {
      // Validate hospital selection for sustain_admin and central_admin
      if ((user?.role === "sustain_admin" || user?.role === "central_admin") && !selectedLocationId) {
        toast({
          variant: "destructive",
          title: "Eksik Bilgi",
          description: "Lütfen hastane seçin"
        });
        return;
      }

      // Validate form
      if (!year || !month || !startDate || !endDate || !amountInteger) {
        toast({
          variant: "destructive",
          title: "Eksik Bilgi",
          description: "Lütfen tüm zorunlu alanları doldurun"
        });
        return;
      }

      // File is required for new entries, optional for edits
      if (!editingEmissionId && !uploadedFile) {
        toast({
          variant: "destructive",
          title: "Eksik Bilgi",
          description: "Lütfen fatura dosyası yükleyin"
        });
        return;
      }

      // Combine integer and decimal parts with Turkish format
      const combinedAmount = amountDecimal 
        ? `${amountInteger},${amountDecimal}`
        : amountInteger;

      // Parse Turkish formatted amount to standard number
      const parsedAmount = parseTurkishNumber(combinedAmount);

      // Get selected supplier and emission factor based on source type
      const currentSupplier = isElectricity ? electricitySupplier : naturalGasSupplier;
      const suppliers = isElectricity ? electricitySuppliers : naturalGasSuppliers;
      const selectedSupplier = suppliers.find(s => s.value === currentSupplier);
      const emissionFactor = selectedSupplier?.factor || "0";

      // Create period string in YYYY-MM format for sorting
      const paddedMonth = month.padStart(2, '0');
      const period = `${year}-${paddedMonth}`;

      // Prepare emission data
      const emissionData: any = {
        locationId: user.role === 'sustain_admin' || user.role === 'central_admin' ? selectedLocationId : user.locationId,
        userId: user.id,
        scope: params.scope,
        categoryId: source.categoryId,
        categoryName: source.categoryName,
        sourceName: source.name,
        sourceUnit: source.unit,
        period,
        startDate: new Date(startDate).toISOString(),
        endDate: new Date(endDate).toISOString(),
        amount: parsedAmount.toFixed(2), // Always store with 2 decimal places
        emissionFactor,
        // Backend will calculate co2Equivalent
        notes: notes || undefined,
      };

      // Add supplier location based on source type
      if (isElectricity) {
        emissionData.electricitySupplierLocation = electricitySupplier;
      } else if (isNaturalGas) {
        emissionData.naturalGasSupplierLocation = naturalGasSupplier;
      }

      // Add file URL if a new file was uploaded
      if (uploadedFile) {
        // TODO: Upload file to storage and get URL
        // For now, we'll use a placeholder
        emissionData.uploadedFile = `/uploads/${uploadedFile.name}`;
      }

      if (editingEmissionId) {
        // Update existing emission
        updateMutation.mutate({ id: editingEmissionId, data: emissionData });
      } else {
        // Create new emission
        createMutation.mutate(emissionData);
      }
    }
  };

  const handleEdit = (emission: SustainabilityEmission) => {
    setEditingEmissionId(emission.id);
    setShowForm(true);
    
    // Fill form with existing data
    if (user?.role === "sustain_admin") {
      setSelectedLocationId(emission.locationId || "");
    }
    
    // Set supplier based on source type
    if (isElectricity) {
      setElectricitySupplier(emission.electricitySupplierLocation || "");
    } else if (isNaturalGas) {
      setNaturalGasSupplier(emission.naturalGasSupplierLocation || "");
    }
    
    // Parse period (YYYY-MM format) to year and month
    if (emission.period) {
      const [periodYear, periodMonth] = emission.period.split('-');
      setYear(periodYear || currentYear.toString());
      setMonth(periodMonth ? parseInt(periodMonth).toString() : "");
    }
    
    // Parse dates
    if (emission.startDate) {
      const startDateObj = new Date(emission.startDate);
      setStartDate(startDateObj.toISOString().split('T')[0]);
    }
    if (emission.endDate) {
      const endDateObj = new Date(emission.endDate);
      setEndDate(endDateObj.toISOString().split('T')[0]);
    }
    
    // Parse amount with Turkish format
    if (emission.amount) {
      const amountStr = emission.amount.toString();
      if (amountStr.includes('.')) {
        const [integer, decimal] = amountStr.split('.');
        setAmountInteger(integer);
        setAmountDecimal(decimal);
      } else {
        setAmountInteger(amountStr);
        setAmountDecimal("");
      }
    }
    
    setNotes(emission.notes || "");
    // Note: We can't restore the uploaded file, but we keep the URL
  };

  const handleDelete = (id: string) => {
    if (confirm("Bu veriyi silmek istediğinizden emin misiniz?")) {
      deleteMutation.mutate(id);
    }
  };

  return (
    <div className="container mx-auto py-6 px-4">
      <div className="mb-6">
        <Button
          variant="ghost"
          size="sm"
          onClick={() => setLocation('/sustainability/data-entry')}
          className="mb-4"
          data-testid="button-back"
        >
          <ArrowLeft className="h-4 w-4 mr-2" />
          Geri
        </Button>
        
        <div className="flex items-start justify-between">
          <div>
            <h1 className="text-2xl font-bold text-gray-900 dark:text-gray-100">{source.name}</h1>
            <p className="text-gray-600 dark:text-gray-400 mt-2">{source.categoryName}</p>
          </div>
          <Badge variant="outline" className="ml-4">
            Birim: {source.unit}
          </Badge>
        </div>
      </div>

      <div className="grid grid-cols-1 gap-6">
        {/* Electricity/Natural Gas Form (Inline) */}
        {(isElectricity || isNaturalGas) && showForm && (
          <Card className="border-blue-200 dark:border-blue-800">
            <CardHeader>
              <div className="flex items-center justify-between">
                <CardTitle>
                  {editingEmissionId 
                    ? `${source.name} Kaydını Düzenle` 
                    : `Yeni ${source.name} Girişi`}
                </CardTitle>
                <Button
                  variant="ghost"
                  size="sm"
                  onClick={resetForm}
                  data-testid="button-close-form"
                >
                  <X className="h-4 w-4" />
                </Button>
              </div>
            </CardHeader>
            <CardContent>
              <form onSubmit={handleSubmit} className="space-y-4">
                {/* Hospital Selection - Only for sustain_admin */}
                {user?.role === "sustain_admin" && (
                  <div className="space-y-2 pb-4 border-b border-gray-200 dark:border-gray-800">
                    <Label htmlFor="hospital">Hastane Seçimi *</Label>
                    <Select value={selectedLocationId} onValueChange={setSelectedLocationId}>
                      <SelectTrigger data-testid="select-hospital">
                        <SelectValue placeholder="Hastane seçin..." />
                      </SelectTrigger>
                      <SelectContent>
                        {hospitals.map((hospital) => (
                          <SelectItem key={hospital.id} value={hospital.id}>
                            {hospital.name}
                          </SelectItem>
                        ))}
                      </SelectContent>
                    </Select>
                  </div>
                )}

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {/* Supplier Selection (Electricity or Natural Gas) */}
                  <div className="space-y-2">
                    <Label htmlFor="supplier">
                      {isElectricity ? "Konum Temelli Elektrik Tedariği *" : "Doğalgaz Tedarikçisi *"}
                    </Label>
                    <Select 
                      value={isElectricity ? electricitySupplier : naturalGasSupplier} 
                      onValueChange={isElectricity ? setElectricitySupplier : setNaturalGasSupplier}
                    >
                      <SelectTrigger data-testid="select-supplier">
                        <SelectValue placeholder={isElectricity ? "Elektrik tedarikçisi seçin..." : "Doğalgaz tedarikçisi seçin..."} />
                      </SelectTrigger>
                      <SelectContent>
                        {(isElectricity ? electricitySuppliers : naturalGasSuppliers).map((supplier) => (
                          <SelectItem key={supplier.value} value={supplier.value}>
                            {supplier.label} ({supplier.factor} {supplier.unit})
                          </SelectItem>
                        ))}
                      </SelectContent>
                    </Select>
                  </div>

                  {/* Year */}
                  <div className="space-y-2">
                    <Label htmlFor="year">Yıl *</Label>
                    <Select value={year} onValueChange={setYear}>
                      <SelectTrigger data-testid="select-year">
                        <SelectValue placeholder="Yıl seçin" />
                      </SelectTrigger>
                      <SelectContent>
                        {years.map((y) => (
                          <SelectItem key={y} value={y.toString()}>
                            {y}
                          </SelectItem>
                        ))}
                      </SelectContent>
                    </Select>
                  </div>

                  {/* Month */}
                  <div className="space-y-2">
                    <Label htmlFor="month">Ay *</Label>
                    <Select value={month} onValueChange={setMonth}>
                      <SelectTrigger data-testid="select-month">
                        <SelectValue placeholder="Ay seçin" />
                      </SelectTrigger>
                      <SelectContent>
                        {months.map((m) => (
                          <SelectItem key={m.value} value={m.value}>
                            {m.label}
                          </SelectItem>
                        ))}
                      </SelectContent>
                    </Select>
                  </div>

                  {/* Start Date */}
                  <div className="space-y-2">
                    <Label htmlFor="startDate">Başlangıç (İlk Okuma) Tarihi *</Label>
                    <Input
                      id="startDate"
                      type="date"
                      value={startDate}
                      onChange={(e) => setStartDate(e.target.value)}
                      data-testid="input-start-date"
                    />
                  </div>

                  {/* End Date */}
                  <div className="space-y-2">
                    <Label htmlFor="endDate">Bitiş (Son Okuma) Tarihi *</Label>
                    <Input
                      id="endDate"
                      type="date"
                      value={endDate}
                      onChange={(e) => setEndDate(e.target.value)}
                      data-testid="input-end-date"
                    />
                  </div>

                  {/* Amount - Split into integer and decimal parts */}
                  <div className="space-y-2">
                    <Label>Tüketim Miktarı ({source.unit}) *</Label>
                    <div className="flex items-center gap-2">
                      <Input
                        id="amountInteger"
                        type="text"
                        inputMode="numeric"
                        placeholder="150"
                        value={amountInteger}
                        onChange={(e) => {
                          let value = e.target.value.replace(/[^\d]/g, ''); // Only digits
                          // Add thousand separators (dots)
                          if (value) {
                            value = value.replace(/\B(?=(\d{3})+(?!\d))/g, '.');
                          }
                          setAmountInteger(value);
                        }}
                        data-testid="input-amount-integer"
                        className="flex-1"
                      />
                      <span className="text-2xl font-bold text-gray-400 dark:text-gray-400">,</span>
                      <Input
                        id="amountDecimal"
                        type="text"
                        inputMode="numeric"
                        placeholder="65"
                        maxLength={2}
                        value={amountDecimal}
                        onChange={(e) => {
                          const value = e.target.value.replace(/[^\d]/g, '').slice(0, 2); // Only 2 digits
                          setAmountDecimal(value);
                        }}
                        data-testid="input-amount-decimal"
                        className="flex-1"
                      />
                    </div>
                    <p className="text-xs text-gray-500 dark:text-gray-500">
                      Örnek: Tam kısım: <strong>1.234</strong> (binlik ayırıcı otomatik), Ondalık kısım: <strong>56</strong> → Sonuç: 1.234,56
                    </p>
                  </div>

                  {/* Calculated Emission - Show only when calculated */}
                  {calculatedEmission > 0 && (
                    <div className="space-y-2">
                      <Label>Hesaplanan CO₂ Emisyonu</Label>
                      <div className="p-3 bg-green-50 dark:bg-green-900/20 border border-green-200 dark:border-green-800 rounded-md">
                        <p className="text-lg font-semibold text-green-700 dark:text-green-400">
                          {formatTurkishNumber(calculatedEmission, 2)} kgCO₂e
                        </p>
                        <p className="text-xs text-gray-600 dark:text-gray-400 mt-1">
                          {amountInteger}{amountDecimal ? `,${amountDecimal}` : ''} {source.unit} × {
                            (isElectricity ? electricitySuppliers : naturalGasSuppliers).find(
                              s => s.value === (isElectricity ? electricitySupplier : naturalGasSupplier)
                            )?.factor || 0
                          }
                        </p>
                      </div>
                    </div>
                  )}

                  {/* File Upload */}
                  <div className="space-y-2">
                    <Label htmlFor="file">Dosya Yükleme (PDF, PNG, JPEG) *</Label>
                    <div className="flex items-center gap-2">
                      <Input
                        id="file"
                        type="file"
                        accept=".pdf,.png,.jpeg,.jpg"
                        onChange={handleFileChange}
                        className="flex-1"
                        data-testid="input-file"
                      />
                      {uploadedFile && (
                        <Badge variant="secondary" className="flex items-center gap-1">
                          <Upload className="h-3 w-3" />
                          {uploadedFile.name}
                        </Badge>
                      )}
                    </div>
                  </div>
                </div>

                {/* Notes - Full Width */}
                <div className="space-y-2">
                  <Label htmlFor="notes">Açıklama</Label>
                  <Textarea
                    id="notes"
                    placeholder="Ek bilgi ve notlar..."
                    value={notes}
                    onChange={(e) => setNotes(e.target.value)}
                    rows={3}
                    data-testid="textarea-notes"
                  />
                </div>

                {/* Action Buttons */}
                <div className="flex justify-end gap-2 pt-4 border-t">
                  <Button
                    type="button"
                    variant="outline"
                    onClick={resetForm}
                    data-testid="button-cancel"
                  >
                    İptal
                  </Button>
                  <Button 
                    type="submit" 
                    disabled={createMutation.isPending || updateMutation.isPending}
                    data-testid="button-save"
                  >
                    {editingEmissionId 
                      ? (updateMutation.isPending ? "Güncelleniyor..." : "Güncelle")
                      : (createMutation.isPending ? "Kaydediliyor..." : "Kaydet")
                    }
                  </Button>
                </div>
              </form>
            </CardContent>
          </Card>
        )}

        {/* Data List */}
        <Card>
          <CardHeader className="flex flex-row items-center justify-between">
            <div>
              <CardTitle className="flex items-center gap-2">
                <Calendar className="h-5 w-5" />
                Girilen Veriler
              </CardTitle>
              <CardDescription>
                {source.name} için kaydedilmiş tüketim verileri
                {user?.role === "sustain_admin" && emissions.length > 0 && (
                  <span> - Tüm Hastaneler ({emissions.length} kayıt)</span>
                )}
              </CardDescription>
            </div>
            {!showForm && (
              <Button 
                onClick={() => setShowForm(true)} 
                data-testid="button-new-entry"
              >
                <Plus className="h-4 w-4 mr-2" />
                Yeni Giriş
              </Button>
            )}
          </CardHeader>
          <CardContent>
            {isLoading ? (
              <div className="text-center py-8 text-gray-500 dark:text-gray-500">Yükleniyor...</div>
            ) : emissions.length === 0 ? (
              <div className="text-center py-12">
                <Calendar className="h-12 w-12 text-gray-300 dark:text-gray-700 mx-auto mb-4" />
                <p className="text-gray-500 dark:text-gray-500 mb-4">Henüz veri girişi yapılmamış</p>
                {!showForm && (
                  <Button 
                    onClick={() => setShowForm(true)}
                    data-testid="button-add-first"
                  >
                    <Plus className="h-4 w-4 mr-2" />
                    İlk Veriyi Ekle
                  </Button>
                )}
              </div>
            ) : (
              <div className="space-y-4">
                {/* Year-based organization for all users */}
                {(() => {
                  // Group emissions by year
                  const groupedByYear = emissions.reduce((acc, emission) => {
                    let year: string;
                    if (emission.period) {
                      year = emission.period.substring(0, 4);
                    } else if (emission.month) {
                      year = new Date(emission.month).getFullYear().toString();
                    } else {
                      year = "Tarih Yok";
                    }
                    
                    if (!acc[year]) {
                      acc[year] = [];
                    }
                    acc[year].push(emission);
                    return acc;
                  }, {} as Record<string, typeof emissions>);

                  const sortedYears = Object.keys(groupedByYear).sort((a, b) => b.localeCompare(a));

                  // Step 1: Show year cards if no year is selected
                  if (!selectedYear) {
                    return (
                      <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
                        {sortedYears.map((year) => {
                          const yearEmissions = groupedByYear[year];
                          const yearTotalCO2 = yearEmissions.reduce((sum, e) => sum + parseFloat(e.co2Equivalent || "0"), 0);
                          const hospitalCount = new Set(yearEmissions.map(e => e.locationId)).size;
                          
                          return (
                            <Card 
                              key={year} 
                              className="hover:shadow-lg transition-all cursor-pointer border-l-4 border-l-purple-500"
                              onClick={() => setSelectedYear(year)}
                              data-testid={`year-card-${year}`}
                            >
                              <CardHeader className="pb-3">
                                <div className="flex items-center gap-2 mb-2">
                                  <Calendar className="h-5 w-5 text-purple-600" />
                                  <CardTitle className="text-2xl font-bold">{year}</CardTitle>
                                </div>
                                <CardDescription className="space-y-1">
                                  <div className="flex items-center gap-2">
                                    <Badge variant="outline">{yearEmissions.length} kayıt</Badge>
                                  </div>
                                  {user?.role === "sustain_admin" && (
                                    <div className="flex items-center gap-2">
                                      <Building2 className="h-3 w-3" />
                                      <span className="text-xs">{hospitalCount} hastane</span>
                                    </div>
                                  )}
                                  <div className="text-sm font-semibold text-green-600 mt-2">
                                    {formatTurkishNumber(yearTotalCO2.toFixed(2))} kgCO₂e
                                  </div>
                                </CardDescription>
                              </CardHeader>
                            </Card>
                          );
                        })}
                      </div>
                    );
                  }

                  // Year is selected
                  const yearEmissions = groupedByYear[selectedYear];
                  
                  // For non-admin users: Skip hospital selection, show data directly
                  if (user?.role !== 'sustain_admin') {
                    // Auto-select hospital for normal users (they only have one)
                    const selectedHospitalEmissions = yearEmissions;
                    
                    return (
                      <div className="space-y-4">
                        <div className="flex items-center justify-between mb-4">
                          <div className="flex items-center gap-2">
                            <Button
                              variant="ghost"
                              size="sm"
                              onClick={() => setSelectedYear(null)}
                              data-testid="button-back-to-years"
                            >
                              <ArrowLeft className="h-4 w-4 mr-2" />
                              Yıllara Dön
                            </Button>
                            <div className="flex items-center gap-2">
                              <Calendar className="h-5 w-5 text-purple-600" />
                              <h3 className="text-xl font-bold">{selectedYear}</h3>
                            </div>
                          </div>
                          <Button
                            variant="outline"
                            size="sm"
                            onClick={() => exportToCSV(selectedHospitalEmissions)}
                            className="flex items-center gap-2"
                            data-testid="button-export-csv"
                          >
                            <Download className="h-4 w-4" />
                            CSV Export
                          </Button>
                        </div>
                        <div className="space-y-3">
                          {selectedHospitalEmissions
                            .sort((a, b) => {
                              const periodA = a.period || a.month || "";
                              const periodB = b.period || b.month || "";
                              return periodB.toString().localeCompare(periodA.toString());
                            })
                            .map((emission) => (
                              <Card key={emission.id} className="border bg-gray-50 dark:bg-gray-800">
                                <CardContent className="p-4">
                                  <div className="flex items-start justify-between">
                                    <div className="flex-1 grid grid-cols-1 md:grid-cols-4 gap-4">
                                      {isElectricity ? (
                                        <>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Dönem</p>
                                            <Badge variant="outline">{formatPeriod(emission.period || "")}</Badge>
                                          </div>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Tedarikçi</p>
                                            <p className="text-sm font-medium">{emission.electricitySupplierLocation}</p>
                                          </div>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Tüketim</p>
                                            <p className="text-sm font-semibold">{formatTurkishNumber(emission.amount)} {emission.sourceUnit}</p>
                                          </div>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">CO₂ Emisyonu</p>
                                            <p className="text-sm font-semibold text-green-600">
                                              {formatTurkishNumber(parseFloat(emission.co2Equivalent || "0").toFixed(2))} kgCO₂e
                                            </p>
                                          </div>
                                        </>
                                      ) : (
                                        <>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Dönem</p>
                                            <Badge variant="outline">
                                              {emission.month && format(new Date(emission.month), "MMMM yyyy")}
                                            </Badge>
                                          </div>
                                          <div className="md:col-span-3">
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Miktar</p>
                                            <p className="text-sm font-semibold">{formatTurkishNumber(emission.amount)} {emission.sourceUnit}</p>
                                          </div>
                                        </>
                                      )}
                                    </div>
                                    <div className="flex gap-2">
                                      <Button
                                        variant="ghost"
                                        size="sm"
                                        onClick={() => handleEdit(emission)}
                                        className="text-blue-600 hover:text-blue-700 hover:bg-blue-50 dark:bg-blue-900/20"
                                        data-testid={`button-edit-${emission.id}`}
                                      >
                                        <Edit className="h-4 w-4" />
                                      </Button>
                                      <Button
                                        variant="ghost"
                                        size="sm"
                                        onClick={() => handleDelete(emission.id)}
                                        className="text-red-600 hover:text-red-700 hover:bg-red-50 dark:bg-red-900/20"
                                        data-testid={`button-delete-${emission.id}`}
                                      >
                                        <Trash2 className="h-4 w-4" />
                                      </Button>
                                    </div>
                                  </div>
                                </CardContent>
                              </Card>
                            ))}
                        </div>
                      </div>
                    );
                  }
                  
                  // For sustain_admin: Show hospital selection step
                  const groupedByHospital = yearEmissions.reduce((acc, emission) => {
                    const locationId = emission.locationId;
                    if (!acc[locationId]) {
                      acc[locationId] = [];
                    }
                    acc[locationId].push(emission);
                    return acc;
                  }, {} as Record<string, typeof emissions>);

                    if (!selectedHospital) {
                      // Filter hospitals based on search
                      const filteredHospitals = Object.entries(groupedByHospital).filter(([locationId]) => {
                        const hospital = hospitals.find(h => h.id === locationId);
                        const hospitalName = hospital?.name || "";
                        return hospitalName.toLowerCase().includes(hospitalSearch.toLowerCase());
                      });

                      return (
                        <div className="space-y-4">
                          <div className="flex items-center justify-between mb-4">
                            <div className="flex items-center gap-2">
                              <Button
                                variant="ghost"
                                size="sm"
                                onClick={() => {
                                  setSelectedYear(null);
                                  setSelectedHospital(null);
                                  setHospitalSearch("");
                                }}
                                data-testid="button-back-to-years"
                              >
                                <ArrowLeft className="h-4 w-4 mr-2" />
                                Yıllara Dön
                              </Button>
                              <div className="flex items-center gap-2">
                                <Calendar className="h-5 w-5 text-purple-600" />
                                <h3 className="text-xl font-bold">{selectedYear}</h3>
                              </div>
                            </div>
                            <Button
                              variant="outline"
                              size="sm"
                              onClick={() => exportToCSV(yearEmissions)}
                              className="flex items-center gap-2"
                              data-testid="button-export-year-csv"
                            >
                              <Download className="h-4 w-4" />
                              CSV Export ({selectedYear})
                            </Button>
                          </div>
                          
                          {/* Search Bar */}
                          <div className="relative">
                            <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 h-4 w-4 text-gray-400 dark:text-gray-400" />
                            <Input
                              type="text"
                              placeholder="Hastane ara..."
                              value={hospitalSearch}
                              onChange={(e) => setHospitalSearch(e.target.value)}
                              className="pl-10"
                              data-testid="input-hospital-search"
                            />
                          </div>

                          {filteredHospitals.length === 0 ? (
                            <div className="text-center py-12">
                              <Building2 className="h-12 w-12 text-gray-300 dark:text-gray-700 mx-auto mb-4" />
                              <p className="text-gray-500 dark:text-gray-500">Arama kriterlerine uygun hastane bulunamadı</p>
                            </div>
                          ) : (
                            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                              {filteredHospitals.map(([locationId, hospitalEmissions]) => {
                                const hospital = hospitals.find(h => h.id === locationId);
                                const hospitalName = hospital?.name || "Bilinmeyen Hastane";
                                const hospitalTotalCO2 = hospitalEmissions.reduce((sum, e) => sum + parseFloat(e.co2Equivalent || "0"), 0);
                                
                                return (
                                  <Card 
                                    key={locationId}
                                    className="hover:shadow-lg transition-all cursor-pointer border-l-4 border-l-blue-500"
                                    onClick={() => setSelectedHospital(locationId)}
                                    data-testid={`hospital-card-${locationId}`}
                                  >
                                    <CardHeader>
                                      <div className="flex items-center gap-2 mb-2">
                                        <Building2 className="h-5 w-5 text-blue-600" />
                                        <CardTitle className="text-lg">{hospitalName}</CardTitle>
                                      </div>
                                      <CardDescription className="space-y-1">
                                        <div>
                                          <Badge variant="outline">{hospitalEmissions.length} kayıt</Badge>
                                        </div>
                                        <div className="text-sm font-semibold text-green-600 mt-2">
                                          {formatTurkishNumber(hospitalTotalCO2.toFixed(2))} kgCO₂e
                                        </div>
                                      </CardDescription>
                                    </CardHeader>
                                  </Card>
                                );
                              })}
                            </div>
                          )}
                        </div>
                      );
                    }

                    // Step 3: Show data for selected hospital
                    const hospitalEmissions = groupedByHospital[selectedHospital];
                    const hospital = hospitals.find(h => h.id === selectedHospital);
                    const hospitalName = hospital?.name || "Bilinmeyen Hastane";
                    
                    return (
                      <div className="space-y-4">
                        <div className="flex items-center justify-between mb-4">
                          <div className="flex items-center gap-2">
                            <Button
                              variant="ghost"
                              size="sm"
                              onClick={() => setSelectedHospital(null)}
                              data-testid="button-back-to-hospitals"
                            >
                              <ArrowLeft className="h-4 w-4 mr-2" />
                              Hastanelere Dön
                            </Button>
                            <div className="flex items-center gap-3">
                              <Calendar className="h-5 w-5 text-purple-600" />
                              <span className="text-lg font-bold">{selectedYear}</span>
                              <span className="text-gray-400 dark:text-gray-400">/</span>
                              <Building2 className="h-5 w-5 text-blue-600" />
                              <span className="text-lg font-bold">{hospitalName}</span>
                            </div>
                          </div>
                          <Button
                            variant="outline"
                            size="sm"
                            onClick={() => exportToCSV(hospitalEmissions)}
                            className="flex items-center gap-2"
                            data-testid="button-export-csv"
                          >
                            <Download className="h-4 w-4" />
                            CSV Export
                          </Button>
                        </div>
                        <div className="space-y-3">
                          {hospitalEmissions
                            .sort((a, b) => {
                              // Sort by period/month descending (most recent first)
                              const periodA = a.period || a.month || "";
                              const periodB = b.period || b.month || "";
                              return periodB.toString().localeCompare(periodA.toString());
                            })
                            .map((emission) => (
                              <Card key={emission.id} className="border bg-gray-50 dark:bg-gray-800">
                                <CardContent className="p-4">
                                  <div className="flex items-start justify-between">
                                    <div className="flex-1 grid grid-cols-1 md:grid-cols-4 gap-4">
                                      {isElectricity ? (
                                        <>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Dönem</p>
                                            <Badge variant="outline">{formatPeriod(emission.period || "")}</Badge>
                                          </div>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Tedarikçi</p>
                                            <p className="text-sm font-medium">{emission.electricitySupplierLocation}</p>
                                          </div>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Tüketim</p>
                                            <p className="text-sm font-semibold">{formatTurkishNumber(emission.amount)} {emission.sourceUnit}</p>
                                          </div>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">CO₂ Emisyonu</p>
                                            <p className="text-sm font-semibold text-green-600">
                                              {formatTurkishNumber(parseFloat(emission.co2Equivalent || "0").toFixed(2))} kgCO₂e
                                            </p>
                                          </div>
                                        </>
                                      ) : (
                                        <>
                                          <div>
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Dönem</p>
                                            <Badge variant="outline">
                                              {emission.month && format(new Date(emission.month), "MMMM yyyy")}
                                            </Badge>
                                          </div>
                                          <div className="md:col-span-3">
                                            <p className="text-xs text-gray-500 dark:text-gray-500 mb-1">Miktar</p>
                                            <p className="text-sm font-semibold">{formatTurkishNumber(emission.amount)} {emission.sourceUnit}</p>
                                          </div>
                                        </>
                                      )}
                                    </div>
                                    <div className="flex gap-2">
                                      <Button
                                        variant="ghost"
                                        size="sm"
                                        onClick={() => handleEdit(emission)}
                                        className="text-blue-600 hover:text-blue-700 hover:bg-blue-50 dark:bg-blue-900/20"
                                        data-testid={`button-edit-${emission.id}`}
                                      >
                                        <Edit className="h-4 w-4" />
                                      </Button>
                                      <Button
                                        variant="ghost"
                                        size="sm"
                                        onClick={() => handleDelete(emission.id)}
                                        className="text-red-600 hover:text-red-700 hover:bg-red-50 dark:bg-red-900/20"
                                        data-testid={`button-delete-${emission.id}`}
                                      >
                                        <Trash2 className="h-4 w-4" />
                                      </Button>
                                    </div>
                                  </div>
                                </CardContent>
                              </Card>
                            ))}
                        </div>
                      </div>
                    );
                  })()}
              </div>
            )}
          </CardContent>
        </Card>

        {/* Charts & Analytics - For Electricity and Natural Gas */}
        {(isElectricity || isNaturalGas) && emissions.length > 0 && (() => {
          // Find all unique years with data
          const allYears = Array.from(
            new Set(
              emissions
                .map(e => e.period?.split('-')[0])
                .filter((year): year is string => Boolean(year))
            )
          ).sort().reverse(); // Most recent year first
          
          // Determine if we're in filtered mode (showing single year/hospital)
          const isFiltered = selectedYear !== null || selectedHospital !== null;
          
          // Calculate summary card totals - always show ONLY the most recent year (or selected year)
          const summaryYear = selectedYear || allYears[0];
          const summaryEmissions = emissions.filter(e => e.period?.startsWith(summaryYear));
          const totalConsumption = summaryEmissions.reduce((sum, e) => sum + parseFloat(e.amount), 0);
          const totalCO2 = summaryEmissions.reduce((sum, e) => sum + (e.co2Equivalent ? parseFloat(e.co2Equivalent) : 0), 0);
          const uniqueHospitals = new Set(summaryEmissions.map(e => e.locationId).filter(Boolean));
          const countValue = uniqueHospitals.size;
          
          let chartData: any[];
          let displayContext: string;
          
          if (isFiltered) {
            // FILTERED MODE: Show single year/hospital monthly data
            let filteredEmissions = emissions;
            
            // Filter by year if selected
            if (selectedYear) {
              filteredEmissions = filteredEmissions.filter(e => e.period?.startsWith(selectedYear));
            }
            
            // Filter by hospital if selected
            if (selectedHospital) {
              filteredEmissions = filteredEmissions.filter(e => e.locationId === selectedHospital);
            }
            
            // Aggregate filtered emissions by month
            type MonthData = Record<number, { consumption: number; co2: number; count: number }>;
            const monthlyAggregation: MonthData = {};
            
            filteredEmissions.forEach(e => {
              if (!e.period) return;
              const [, monthStr] = e.period.split('-');
              const monthIndex = parseInt(monthStr) - 1; // 0-11
              
              if (!monthlyAggregation[monthIndex]) {
                monthlyAggregation[monthIndex] = { consumption: 0, co2: 0, count: 0 };
              }
              
              monthlyAggregation[monthIndex].consumption += parseFloat(e.amount);
              monthlyAggregation[monthIndex].co2 += e.co2Equivalent ? parseFloat(e.co2Equivalent) : 0;
              monthlyAggregation[monthIndex].count += 1;
            });
            
            // Prepare chart data: one row per month
            chartData = MONTH_LABELS.map((monthLabel, monthIndex) => {
              const data = monthlyAggregation[monthIndex];
              return {
                month: monthLabel,
                monthIndex,
                consumption: data?.consumption || 0,
                co2: data?.co2 || 0,
                color: MONTH_COLORS[monthIndex]
              };
            });
            
            const displayYear = selectedYear || allYears[0];
            const hospital = selectedHospital ? hospitals.find(h => h.id === selectedHospital) : null;
            displayContext = selectedHospital 
              ? `${displayYear} - ${hospital?.name || 'Hastane'}`
              : `${displayYear} - Tüm Hastaneler`;
          } else {
            // MULTI-YEAR MODE: Show all years comparison
            type MonthYearData = Record<number, Record<string, { consumption: number; co2: number; count: number }>>;
            const monthYearAggregation: MonthYearData = {};
            
            emissions.forEach(e => {
              if (!e.period) return;
              const [year, monthStr] = e.period.split('-');
              const monthIndex = parseInt(monthStr) - 1; // 0-11
              
              if (!monthYearAggregation[monthIndex]) {
                monthYearAggregation[monthIndex] = {};
              }
              if (!monthYearAggregation[monthIndex][year]) {
                monthYearAggregation[monthIndex][year] = { consumption: 0, co2: 0, count: 0 };
              }
              
              monthYearAggregation[monthIndex][year].consumption += parseFloat(e.amount);
              monthYearAggregation[monthIndex][year].co2 += e.co2Equivalent ? parseFloat(e.co2Equivalent) : 0;
              monthYearAggregation[monthIndex][year].count += 1;
            });
            
            // Prepare chart data: one row per month with columns for each year
            chartData = MONTH_LABELS.map((monthLabel, monthIndex) => {
              const row: any = {
                month: monthLabel,
                monthIndex,
                color: MONTH_COLORS[monthIndex]
              };
              
              allYears.forEach(year => {
                const data = monthYearAggregation[monthIndex]?.[year];
                row[`${year}_consumption`] = data?.consumption || 0;
                row[`${year}_co2`] = data?.co2 || 0;
              });
              
              return row;
            });
            
            displayContext = "Tüm Yıllar - Karşılaştırma";
          }

          return (
            <>
              {/* Summary Cards - Always show most recent year */}
              <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                <Card>
                  <CardHeader className="pb-2">
                    <CardDescription>{summaryYear} - Toplam Tüketim</CardDescription>
                  </CardHeader>
                  <CardContent>
                    <div className="text-3xl font-bold text-blue-600 dark:text-blue-400">
                      {formatTurkishNumber(totalConsumption)} {source.unit}
                    </div>
                  </CardContent>
                </Card>

                <Card>
                  <CardHeader className="pb-2">
                    <CardDescription>{summaryYear} - Toplam CO₂ Emisyonu</CardDescription>
                  </CardHeader>
                  <CardContent>
                    <div className="text-3xl font-bold text-green-600 dark:text-green-400">
                      {formatTurkishNumber(totalCO2)} kgCO₂e
                    </div>
                  </CardContent>
                </Card>

                <Card>
                  <CardHeader className="pb-2">
                    <CardDescription>Hastane Sayısı</CardDescription>
                  </CardHeader>
                  <CardContent>
                    <div className="text-3xl font-bold text-purple-600 dark:text-purple-400">
                      {countValue}
                    </div>
                  </CardContent>
                </Card>
              </div>

              {/* CO2 Emissions Chart */}
              <Card id="co2-chart-card">
                <CardHeader>
                  <div className="flex items-center justify-between">
                    <div>
                      <CardTitle className="flex items-center gap-2">
                        <TrendingUp className="h-5 w-5" />
                        {isFiltered ? `${displayContext} - Aylara Göre CO₂ Emisyonu` : 'Aylara Göre CO₂ Emisyonu - Yıl Karşılaştırması'}
                      </CardTitle>
                      <CardDescription>
                        {isFiltered ? 'Aylık toplam karbon emisyonu (kgCO₂e)' : 'Tüm hastanelerin aylık toplam karbon emisyonu (kgCO₂e)'}
                      </CardDescription>
                    </div>
                    <Button
                      variant="outline"
                      size="sm"
                      onClick={() => downloadChartAsPNG('co2-chart-card', `CO2-Grafigi-${new Date().toLocaleDateString('tr-TR')}`)}
                      className="flex items-center gap-2"
                      data-testid="button-download-co2-chart"
                    >
                      <Download className="h-4 w-4" />
                      PNG İndir
                    </Button>
                  </div>
                </CardHeader>
                <CardContent>
                  <ResponsiveContainer width="100%" height={400}>
                    <BarChart data={chartData}>
                      <CartesianGrid strokeDasharray="3 3" className="dark:stroke-gray-700" />
                      <XAxis dataKey="month" className="dark:fill-gray-400" />
                      <YAxis className="dark:fill-gray-400" />
                      <Tooltip 
                        formatter={(value: number) => formatTurkishNumber(value) + ' kgCO₂e'}
                        labelStyle={{ color: '#000' }}
                        contentStyle={{ backgroundColor: 'white', border: '1px solid #ccc' }}
                      />
                      <Legend />
                      {isFiltered ? (
                        <Bar 
                          dataKey="co2"
                          name="CO₂ Emisyonu"
                          fill={getYearColorCO2(selectedYear || allYears[0], 0)}
                        />
                      ) : (
                        allYears.map((year, idx) => (
                          <Bar 
                            key={year}
                            dataKey={`${year}_co2`}
                            name={`${year} CO₂`}
                            fill={getYearColorCO2(year, idx)}
                          />
                        ))
                      )}
                    </BarChart>
                  </ResponsiveContainer>
                  
                  {/* Color Customization - CO2 */}
                  <div className="mt-4 pt-4 border-t border-gray-200 dark:border-gray-700">
                    <div className="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">CO₂ Grafiği Renkleri:</div>
                    <div className="flex flex-wrap gap-4">
                      {isFiltered ? (
                        <div className="flex items-center gap-2">
                          <Label htmlFor={`color-co2-${selectedYear || allYears[0]}`} className="text-sm">{selectedYear || allYears[0]}:</Label>
                          <input
                            id={`color-co2-${selectedYear || allYears[0]}`}
                            type="color"
                            value={getYearColorCO2(selectedYear || allYears[0], 0)}
                            onChange={(e) => {
                              const year = selectedYear || allYears[0];
                              updateChartColorCO2(year, e.target.value);
                            }}
                            className="w-12 h-8 rounded border border-gray-300 dark:border-gray-600 cursor-pointer"
                          />
                        </div>
                      ) : (
                        allYears.map((year, idx) => (
                          <div key={year} className="flex items-center gap-2">
                            <Label htmlFor={`color-co2-${year}`} className="text-sm">{year}:</Label>
                            <input
                              id={`color-co2-${year}`}
                              type="color"
                              value={getYearColorCO2(year, idx)}
                              onChange={(e) => updateChartColorCO2(year, e.target.value)}
                              className="w-12 h-8 rounded border border-gray-300 dark:border-gray-600 cursor-pointer"
                            />
                          </div>
                        ))
                      )}
                    </div>
                  </div>
                </CardContent>
              </Card>

              {/* Electricity Consumption Chart */}
              <Card id="consumption-chart-card">
                <CardHeader>
                  <div className="flex items-center justify-between">
                    <div>
                      <CardTitle className="flex items-center gap-2">
                        <BarChart3 className="h-5 w-5" />
                        {isFiltered ? `${displayContext} - Aylara Göre ${source.name}` : `${source.name} - Yıl Karşılaştırması`}
                      </CardTitle>
                      <CardDescription>
                        {isFiltered ? `Aylık toplam ${source.name.toLowerCase()} (${source.unit})` : `Tüm hastanelerin aylık toplam ${source.name.toLowerCase()} (${source.unit})`}
                      </CardDescription>
                    </div>
                    <Button
                      variant="outline"
                      size="sm"
                      onClick={() => downloadChartAsPNG('consumption-chart-card', `Elektrik-Tuketimi-Grafigi-${new Date().toLocaleDateString('tr-TR')}`)}
                      className="flex items-center gap-2"
                      data-testid="button-download-consumption-chart"
                    >
                      <Download className="h-4 w-4" />
                      PNG İndir
                    </Button>
                  </div>
                </CardHeader>
                <CardContent>
                  <ResponsiveContainer width="100%" height={400}>
                    <BarChart data={chartData}>
                      <CartesianGrid strokeDasharray="3 3" className="dark:stroke-gray-700" />
                      <XAxis dataKey="month" className="dark:fill-gray-400" />
                      <YAxis className="dark:fill-gray-400" />
                      <Tooltip 
                        formatter={(value: number) => formatTurkishNumber(value) + ' ' + source.unit}
                        labelStyle={{ color: '#000' }}
                        contentStyle={{ backgroundColor: 'white', border: '1px solid #ccc' }}
                      />
                      <Legend />
                      {isFiltered ? (
                        <Bar 
                          dataKey="consumption"
                          name="Elektrik Tüketimi"
                          fill={getYearColorConsumption(selectedYear || allYears[0], 0)}
                        />
                      ) : (
                        allYears.map((year, idx) => (
                          <Bar 
                            key={year}
                            dataKey={`${year}_consumption`}
                            name={`${year} Tüketim`}
                            fill={getYearColorConsumption(year, idx)}
                          />
                        ))
                      )}
                    </BarChart>
                  </ResponsiveContainer>
                  
                  {/* Color Customization - Consumption */}
                  <div className="mt-4 pt-4 border-t border-gray-200 dark:border-gray-700">
                    <div className="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">Tüketim Grafiği Renkleri:</div>
                    <div className="flex flex-wrap gap-4">
                      {isFiltered ? (
                        <div className="flex items-center gap-2">
                          <Label htmlFor={`color-consumption-${selectedYear || allYears[0]}`} className="text-sm">{selectedYear || allYears[0]}:</Label>
                          <input
                            id={`color-consumption-${selectedYear || allYears[0]}`}
                            type="color"
                            value={getYearColorConsumption(selectedYear || allYears[0], 0)}
                            onChange={(e) => {
                              const year = selectedYear || allYears[0];
                              updateChartColorConsumption(year, e.target.value);
                            }}
                            className="w-12 h-8 rounded border border-gray-300 dark:border-gray-600 cursor-pointer"
                          />
                        </div>
                      ) : (
                        allYears.map((year, idx) => (
                          <div key={year} className="flex items-center gap-2">
                            <Label htmlFor={`color-consumption-${year}`} className="text-sm">{year}:</Label>
                            <input
                              id={`color-consumption-${year}`}
                              type="color"
                              value={getYearColorConsumption(year, idx)}
                              onChange={(e) => updateChartColorConsumption(year, e.target.value)}
                              className="w-12 h-8 rounded border border-gray-300 dark:border-gray-600 cursor-pointer"
                            />
                          </div>
                        ))
                      )}
                    </div>
                  </div>
                </CardContent>
              </Card>
            </>
          );
        })()}
      </div>
    </div>
  );
}
```

---

### 6.4 sustainability-parameters.tsx

```tsx
import { useState } from "react";
import type { FormEvent } from "react";
import { useQuery, useMutation } from "@tanstack/react-query";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { useToast } from "@/hooks/use-toast";
import { Settings, Plus, Pencil, Trash2, Loader2 } from "lucide-react";
import { apiRequest, queryClient } from "@/lib/queryClient";
import type { EmissionParameter } from "@shared/schema";

export default function SustainabilityParameters() {
  const { toast } = useToast();
  const [isCreateDialogOpen, setIsCreateDialogOpen] = useState(false);
  const [isEditDialogOpen, setIsEditDialogOpen] = useState(false);
  const [isDeleteDialogOpen, setIsDeleteDialogOpen] = useState(false);
  const [selectedParameter, setSelectedParameter] = useState<EmissionParameter | null>(null);
  const [searchTerm, setSearchTerm] = useState("");

  const [formData, setFormData] = useState({
    category: "",
    title: "",
    emissionSource: "",
    emissionFactor: "",
    unit: "",
    source: "",
  });

  const { data: parameters = [], isLoading } = useQuery<EmissionParameter[]>({
    queryKey: ["/api/emission-parameters"],
  });

  const createMutation = useMutation({
    mutationFn: async (data: typeof formData) => {
      return apiRequest("POST", "/api/emission-parameters", data);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["/api/emission-parameters"] });
      setIsCreateDialogOpen(false);
      resetForm();
      toast({
        title: "Başarılı",
        description: "Emisyon parametresi başarıyla oluşturuldu",
      });
    },
    onError: (error: any) => {
      toast({
        title: "Hata",
        description: error.message || "Parametre oluşturulurken bir hata oluştu",
        variant: "destructive",
      });
    },
  });

  const updateMutation = useMutation({
    mutationFn: async ({ id, data }: { id: string; data: typeof formData }) => {
      return apiRequest("PATCH", `/api/emission-parameters/${id}`, data);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["/api/emission-parameters"] });
      setIsEditDialogOpen(false);
      setSelectedParameter(null);
      resetForm();
      toast({
        title: "Başarılı",
        description: "Emisyon parametresi başarıyla güncellendi",
      });
    },
    onError: (error: any) => {
      toast({
        title: "Hata",
        description: error.message || "Parametre güncellenirken bir hata oluştu",
        variant: "destructive",
      });
    },
  });

  const deleteMutation = useMutation({
    mutationFn: async (id: string) => {
      return apiRequest("DELETE", `/api/emission-parameters/${id}`);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["/api/emission-parameters"] });
      setIsDeleteDialogOpen(false);
      setSelectedParameter(null);
      toast({
        title: "Başarılı",
        description: "Emisyon parametresi başarıyla silindi",
      });
    },
    onError: (error: any) => {
      toast({
        title: "Hata",
        description: error.message || "Parametre silinirken bir hata oluştu",
        variant: "destructive",
      });
    },
  });

  const resetForm = () => {
    setFormData({
      category: "",
      title: "",
      emissionSource: "",
      emissionFactor: "",
      unit: "",
      source: "",
    });
  };

  const handleCreate = () => {
    resetForm();
    setIsCreateDialogOpen(true);
  };

  const handleEdit = (parameter: EmissionParameter) => {
    setSelectedParameter(parameter);
    setFormData({
      category: parameter.category,
      title: parameter.title,
      emissionSource: parameter.emissionSource,
      emissionFactor: parameter.emissionFactor,
      unit: parameter.unit,
      source: parameter.source,
    });
    setIsEditDialogOpen(true);
  };

  const handleDelete = (parameter: EmissionParameter) => {
    setSelectedParameter(parameter);
    setIsDeleteDialogOpen(true);
  };

  const handleSubmitCreate = (e: FormEvent) => {
    e.preventDefault();
    createMutation.mutate(formData);
  };

  const handleSubmitEdit = (e: FormEvent) => {
    e.preventDefault();
    if (selectedParameter) {
      updateMutation.mutate({ id: selectedParameter.id, data: formData });
    }
  };

  const handleConfirmDelete = () => {
    if (selectedParameter) {
      deleteMutation.mutate(selectedParameter.id);
    }
  };

  const filteredParameters = parameters.filter(
    (param) =>
      param.category.toLowerCase().includes(searchTerm.toLowerCase()) ||
      param.title.toLowerCase().includes(searchTerm.toLowerCase()) ||
      param.emissionSource.toLowerCase().includes(searchTerm.toLowerCase())
  );

  return (
    <div className="container mx-auto p-6">
      <div className="mb-6">
        <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100 flex items-center gap-2">
          <Settings className="w-8 h-8 text-primary" />
          Emisyon Parametreleri
        </h1>
        <p className="text-gray-600 dark:text-gray-400 mt-2">
          Sürdürülebilirlik hesaplamalarında kullanılan emisyon faktörleri ve parametreler
        </p>
      </div>

      <Card>
        <CardHeader>
          <div className="flex items-center justify-between">
            <CardTitle>Parametreler Listesi</CardTitle>
            <Button onClick={handleCreate} data-testid="button-create-parameter">
              <Plus className="w-4 h-4 mr-2" />
              Yeni Parametre
            </Button>
          </div>
        </CardHeader>
        <CardContent>
          <div className="mb-4">
            <Input
              placeholder="Kategori, başlık veya emisyon kaynağına göre ara..."
              value={searchTerm}
              onChange={(e) => setSearchTerm(e.target.value)}
              data-testid="input-search-parameters"
            />
          </div>

          {isLoading ? (
            <div className="flex items-center justify-center p-8">
              <Loader2 className="w-8 h-8 animate-spin text-primary" />
            </div>
          ) : (
            <div className="border rounded-md">
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Kategori</TableHead>
                    <TableHead>Başlık</TableHead>
                    <TableHead>Emisyon Kaynağı</TableHead>
                    <TableHead>Emisyon Faktörü</TableHead>
                    <TableHead>Birim</TableHead>
                    <TableHead>Kaynak</TableHead>
                    <TableHead className="text-right">İşlemler</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {filteredParameters.length === 0 ? (
                    <TableRow>
                      <TableCell colSpan={7} className="text-center text-gray-500 dark:text-gray-500 py-8">
                        Parametre bulunamadı
                      </TableCell>
                    </TableRow>
                  ) : (
                    filteredParameters.map((param) => (
                      <TableRow key={param.id} data-testid={`row-parameter-${param.id}`}>
                        <TableCell className="font-medium">{param.category}</TableCell>
                        <TableCell>{param.title}</TableCell>
                        <TableCell>{param.emissionSource}</TableCell>
                        <TableCell>{param.emissionFactor}</TableCell>
                        <TableCell>{param.unit}</TableCell>
                        <TableCell className="text-sm text-gray-600 dark:text-gray-400">{param.source}</TableCell>
                        <TableCell className="text-right">
                          <div className="flex items-center justify-end gap-2">
                            <Button
                              variant="outline"
                              size="sm"
                              onClick={() => handleEdit(param)}
                              data-testid={`button-edit-${param.id}`}
                            >
                              <Pencil className="w-4 h-4" />
                            </Button>
                            <Button
                              variant="outline"
                              size="sm"
                              onClick={() => handleDelete(param)}
                              data-testid={`button-delete-${param.id}`}
                            >
                              <Trash2 className="w-4 h-4 text-red-600" />
                            </Button>
                          </div>
                        </TableCell>
                      </TableRow>
                    ))
                  )}
                </TableBody>
              </Table>
            </div>
          )}

          <div className="mt-4 text-sm text-gray-600 dark:text-gray-400">
            Toplam {filteredParameters.length} parametre gösteriliyor
          </div>
        </CardContent>
      </Card>

      {/* Create Dialog */}
      <Dialog open={isCreateDialogOpen} onOpenChange={setIsCreateDialogOpen}>
        <DialogContent className="max-w-2xl">
          <DialogHeader>
            <DialogTitle>Yeni Emisyon Parametresi</DialogTitle>
            <DialogDescription>
              Yeni bir emisyon parametresi eklemek için aşağıdaki bilgileri girin
            </DialogDescription>
          </DialogHeader>
          <form onSubmit={handleSubmitCreate}>
            <div className="grid gap-4 py-4">
              <div className="grid gap-2">
                <Label htmlFor="category">Kategori</Label>
                <Input
                  id="category"
                  value={formData.category}
                  onChange={(e) => setFormData({ ...formData, category: e.target.value })}
                  placeholder="ör. Kategori 1.1"
                  required
                  data-testid="input-category"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="title">Başlık</Label>
                <Input
                  id="title"
                  value={formData.title}
                  onChange={(e) => setFormData({ ...formData, title: e.target.value })}
                  placeholder="ör. Sabit Yanma"
                  required
                  data-testid="input-title"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="emissionSource">Emisyon Kaynağı</Label>
                <Input
                  id="emissionSource"
                  value={formData.emissionSource}
                  onChange={(e) => setFormData({ ...formData, emissionSource: e.target.value })}
                  placeholder="ör. İstme Amaçlı Doğalgaz - TR"
                  required
                  data-testid="input-emission-source"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="emissionFactor">Emisyon Faktörü</Label>
                <Input
                  id="emissionFactor"
                  value={formData.emissionFactor}
                  onChange={(e) => setFormData({ ...formData, emissionFactor: e.target.value })}
                  placeholder="ör. 1.943"
                  required
                  data-testid="input-emission-factor"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="unit">Birim</Label>
                <Input
                  id="unit"
                  value={formData.unit}
                  onChange={(e) => setFormData({ ...formData, unit: e.target.value })}
                  placeholder="ör. kgCO2e/Sm³"
                  required
                  data-testid="input-unit"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="source">Kaynak</Label>
                <Input
                  id="source"
                  value={formData.source}
                  onChange={(e) => setFormData({ ...formData, source: e.target.value })}
                  placeholder="ör. IPCC Vol.2 Ch.2 Tablo 2.3"
                  required
                  data-testid="input-source"
                />
              </div>
            </div>
            <DialogFooter>
              <Button
                type="button"
                variant="outline"
                onClick={() => setIsCreateDialogOpen(false)}
                disabled={createMutation.isPending}
              >
                İptal
              </Button>
              <Button type="submit" disabled={createMutation.isPending} data-testid="button-submit-create">
                {createMutation.isPending ? (
                  <>
                    <Loader2 className="w-4 h-4 mr-2 animate-spin" />
                    Oluşturuluyor...
                  </>
                ) : (
                  "Oluştur"
                )}
              </Button>
            </DialogFooter>
          </form>
        </DialogContent>
      </Dialog>

      {/* Edit Dialog */}
      <Dialog open={isEditDialogOpen} onOpenChange={setIsEditDialogOpen}>
        <DialogContent className="max-w-2xl">
          <DialogHeader>
            <DialogTitle>Emisyon Parametresi Düzenle</DialogTitle>
            <DialogDescription>
              Emisyon parametresini güncellemek için aşağıdaki bilgileri düzenleyin
            </DialogDescription>
          </DialogHeader>
          <form onSubmit={handleSubmitEdit}>
            <div className="grid gap-4 py-4">
              <div className="grid gap-2">
                <Label htmlFor="edit-category">Kategori</Label>
                <Input
                  id="edit-category"
                  value={formData.category}
                  onChange={(e) => setFormData({ ...formData, category: e.target.value })}
                  required
                  data-testid="input-edit-category"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="edit-title">Başlık</Label>
                <Input
                  id="edit-title"
                  value={formData.title}
                  onChange={(e) => setFormData({ ...formData, title: e.target.value })}
                  required
                  data-testid="input-edit-title"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="edit-emissionSource">Emisyon Kaynağı</Label>
                <Input
                  id="edit-emissionSource"
                  value={formData.emissionSource}
                  onChange={(e) => setFormData({ ...formData, emissionSource: e.target.value })}
                  required
                  data-testid="input-edit-emission-source"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="edit-emissionFactor">Emisyon Faktörü</Label>
                <Input
                  id="edit-emissionFactor"
                  value={formData.emissionFactor}
                  onChange={(e) => setFormData({ ...formData, emissionFactor: e.target.value })}
                  required
                  data-testid="input-edit-emission-factor"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="edit-unit">Birim</Label>
                <Input
                  id="edit-unit"
                  value={formData.unit}
                  onChange={(e) => setFormData({ ...formData, unit: e.target.value })}
                  required
                  data-testid="input-edit-unit"
                />
              </div>
              <div className="grid gap-2">
                <Label htmlFor="edit-source">Kaynak</Label>
                <Input
                  id="edit-source"
                  value={formData.source}
                  onChange={(e) => setFormData({ ...formData, source: e.target.value })}
                  required
                  data-testid="input-edit-source"
                />
              </div>
            </div>
            <DialogFooter>
              <Button
                type="button"
                variant="outline"
                onClick={() => setIsEditDialogOpen(false)}
                disabled={updateMutation.isPending}
              >
                İptal
              </Button>
              <Button type="submit" disabled={updateMutation.isPending} data-testid="button-submit-edit">
                {updateMutation.isPending ? (
                  <>
                    <Loader2 className="w-4 h-4 mr-2 animate-spin" />
                    Güncelleniyor...
                  </>
                ) : (
                  "Güncelle"
                )}
              </Button>
            </DialogFooter>
          </form>
        </DialogContent>
      </Dialog>

      {/* Delete Confirmation Dialog */}
      <AlertDialog open={isDeleteDialogOpen} onOpenChange={setIsDeleteDialogOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Parametreyi Sil</AlertDialogTitle>
            <AlertDialogDescription>
              Bu emisyon parametresini silmek istediğinizden emin misiniz?
              {selectedParameter && (
                <div className="mt-2 p-3 bg-gray-50 dark:bg-gray-800 rounded-md">
                  <p className="font-medium">{selectedParameter.emissionSource}</p>
                  <p className="text-sm text-gray-600 dark:text-gray-400">{selectedParameter.category} - {selectedParameter.title}</p>
                </div>
              )}
              Bu işlem geri alınamaz.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel disabled={deleteMutation.isPending}>İptal</AlertDialogCancel>
            <AlertDialogAction
              onClick={handleConfirmDelete}
              disabled={deleteMutation.isPending}
              className="bg-red-600 hover:bg-red-700"
              data-testid="button-confirm-delete"
            >
              {deleteMutation.isPending ? (
                <>
                  <Loader2 className="w-4 h-4 mr-2 animate-spin" />
                  Siliniyor...
                </>
              ) : (
                "Sil"
              )}
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

---

### 6.5 sustainability-analysis.tsx

```tsx
import { useQuery } from "@tanstack/react-query";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { useAuth } from "@/hooks/useAuth";
import { ArrowLeft, BarChart3, TrendingUp } from "lucide-react";
import { useLocation } from "wouter";
import type { SustainabilityEmission } from "@shared/schema";
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer, LineChart, Line } from "recharts";

export default function SustainabilityAnalysis() {
  const [, setLocation] = useLocation();
  const { user } = useAuth();

  // Fetch all emissions for this location
  const { data: emissions = [], isLoading } = useQuery<SustainabilityEmission[]>({
    queryKey: [`/api/sustainability/emissions?locationId=${user?.locationId}`],
    enabled: !!user?.locationId,
  });

  if (!user) {
    return <div>Yükleniyor...</div>;
  }

  // Group electricity emissions by period
  const electricityEmissions = emissions.filter(e => e.sourceName === "Elektrik Tüketimi");
  
  const electricityByPeriod = electricityEmissions.map(e => ({
    period: e.period || "Bilinmeyen",
    consumption: parseFloat(e.amount),
    co2: e.co2Equivalent ? parseFloat(e.co2Equivalent) : 0,
    supplier: e.electricitySupplierLocation || "",
    startDate: e.startDate,
  })).sort((a, b) => {
    // Sort by period number
    const periodA = parseInt(a.period.split('.')[0]);
    const periodB = parseInt(b.period.split('.')[0]);
    return periodA - periodB;
  });

  // Calculate totals
  const totalConsumption = electricityByPeriod.reduce((sum, item) => sum + item.consumption, 0);
  const totalCO2 = electricityByPeriod.reduce((sum, item) => sum + item.co2, 0);

  // Group by supplier
  const bySupplier: Record<string, { consumption: number; co2: number }> = {};
  electricityEmissions.forEach(e => {
    const supplier = e.electricitySupplierLocation || "Bilinmeyen";
    if (!bySupplier[supplier]) {
      bySupplier[supplier] = { consumption: 0, co2: 0 };
    }
    bySupplier[supplier].consumption += parseFloat(e.amount);
    bySupplier[supplier].co2 += e.co2Equivalent ? parseFloat(e.co2Equivalent) : 0;
  });

  const supplierData = Object.entries(bySupplier).map(([supplier, data]) => ({
    supplier: supplier.replace('Elektrik-', ''),
    consumption: data.consumption,
    co2: data.co2,
  }));

  return (
    <div className="container mx-auto py-6 px-4">
      <div className="mb-6">
        <Button
          variant="ghost"
          size="sm"
          onClick={() => setLocation('/sustainability/data-entry')}
          className="mb-4"
          data-testid="button-back"
        >
          <ArrowLeft className="h-4 w-4 mr-2" />
          Geri
        </Button>
        
        <div className="flex items-start justify-between">
          <div>
            <h1 className="text-2xl font-bold text-gray-900 dark:text-gray-100">Emisyon Analizi</h1>
            <p className="text-gray-600 dark:text-gray-400 mt-2">
              Elektrik tüketimi ve CO₂ emisyon grafikleri
            </p>
          </div>
        </div>
      </div>

      {isLoading ? (
        <div className="text-center py-12">
          <div className="text-gray-500">Veriler yükleniyor...</div>
        </div>
      ) : electricityEmissions.length === 0 ? (
        <Card>
          <CardContent className="py-12 text-center">
            <BarChart3 className="h-16 w-16 text-gray-300 dark:text-gray-700 mx-auto mb-4" />
            <p className="text-gray-500 dark:text-gray-400 mb-4">
              Henüz elektrik tüketimi verisi bulunmamaktadır
            </p>
            <Button onClick={() => setLocation('/sustainability/data-entry')}>
              Veri Girişi Yap
            </Button>
          </CardContent>
        </Card>
      ) : (
        <div className="space-y-6">
          {/* Summary Cards */}
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
            <Card>
              <CardHeader className="pb-2">
                <CardDescription>Toplam Elektrik Tüketimi</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="text-3xl font-bold text-blue-600 dark:text-blue-400">
                  {totalConsumption.toLocaleString('tr-TR', { maximumFractionDigits: 2 })} kWh
                </div>
              </CardContent>
            </Card>

            <Card>
              <CardHeader className="pb-2">
                <CardDescription>Toplam CO₂ Emisyonu</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="text-3xl font-bold text-green-600 dark:text-green-400">
                  {totalCO2.toLocaleString('tr-TR', { maximumFractionDigits: 2 })} kgCO₂e
                </div>
              </CardContent>
            </Card>

            <Card>
              <CardHeader className="pb-2">
                <CardDescription>Dönem Sayısı</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="text-3xl font-bold text-purple-600 dark:text-purple-400">
                  {electricityByPeriod.length}
                </div>
              </CardContent>
            </Card>
          </div>

          {/* CO2 Emissions by Period Chart */}
          <Card>
            <CardHeader>
              <CardTitle className="flex items-center gap-2">
                <TrendingUp className="h-5 w-5" />
                Dönemlere Göre CO₂ Emisyonu
              </CardTitle>
              <CardDescription>
                Her dönem için hesaplanan toplam karbon emisyonu (kgCO₂e)
              </CardDescription>
            </CardHeader>
            <CardContent>
              <ResponsiveContainer width="100%" height={300}>
                <BarChart data={electricityByPeriod}>
                  <CartesianGrid strokeDasharray="3 3" />
                  <XAxis dataKey="period" />
                  <YAxis />
                  <Tooltip 
                    formatter={(value: number) => `${value.toFixed(2)} kgCO₂e`}
                    labelStyle={{ color: '#000' }}
                  />
                  <Legend />
                  <Bar dataKey="co2" fill="#10b981" name="CO₂ Emisyonu (kgCO₂e)" />
                </BarChart>
              </ResponsiveContainer>
            </CardContent>
          </Card>

          {/* Electricity Consumption by Period Chart */}
          <Card>
            <CardHeader>
              <CardTitle className="flex items-center gap-2">
                <BarChart3 className="h-5 w-5" />
                Dönemlere Göre Elektrik Tüketimi
              </CardTitle>
              <CardDescription>
                Her dönem için elektrik tüketim miktarı (kWh)
              </CardDescription>
            </CardHeader>
            <CardContent>
              <ResponsiveContainer width="100%" height={300}>
                <LineChart data={electricityByPeriod}>
                  <CartesianGrid strokeDasharray="3 3" />
                  <XAxis dataKey="period" />
                  <YAxis />
                  <Tooltip 
                    formatter={(value: number) => `${value.toFixed(2)} kWh`}
                    labelStyle={{ color: '#000' }}
                  />
                  <Legend />
                  <Line 
                    type="monotone" 
                    dataKey="consumption" 
                    stroke="#3b82f6" 
                    strokeWidth={2}
                    name="Tüketim (kWh)" 
                  />
                </LineChart>
              </ResponsiveContainer>
            </CardContent>
          </Card>

          {/* Emissions by Supplier Chart */}
          {supplierData.length > 1 && (
            <Card>
              <CardHeader>
                <CardTitle className="flex items-center gap-2">
                  <BarChart3 className="h-5 w-5" />
                  Tedarikçiye Göre CO₂ Emisyonu
                </CardTitle>
                <CardDescription>
                  Farklı elektrik tedarikçilerine göre toplam emisyon
                </CardDescription>
              </CardHeader>
              <CardContent>
                <ResponsiveContainer width="100%" height={300}>
                  <BarChart data={supplierData}>
                    <CartesianGrid strokeDasharray="3 3" />
                    <XAxis dataKey="supplier" />
                    <YAxis />
                    <Tooltip 
                      formatter={(value: number) => `${value.toFixed(2)} kgCO₂e`}
                      labelStyle={{ color: '#000' }}
                    />
                    <Legend />
                    <Bar dataKey="co2" fill="#8b5cf6" name="CO₂ Emisyonu (kgCO₂e)" />
                  </BarChart>
                </ResponsiveContainer>
              </CardContent>
            </Card>
          )}
        </div>
      )}
    </div>
  );
}
```

---

### 6.6 sustainability-analytics.tsx

```tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { BarChart3 } from "lucide-react";

export default function SustainabilityAnalytics() {
  return (
    <div className="container mx-auto p-6">
      <div className="mb-6">
        <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100 flex items-center gap-2">
          <BarChart3 className="w-8 h-8 text-primary" />
          Grafik & Analiz
        </h1>
        <p className="text-gray-600 dark:text-gray-400 mt-2">
          Sürdürülebilirlik verilerinin grafik ve analiz görünümü
        </p>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Grafik & Analiz Modülü</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-gray-600 dark:text-gray-400">
            Bu bölümde sürdürülebilirlik verileri görselleştirilecek ve analiz edilecek.
          </p>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

### 6.7 sustainability-reports.tsx

```tsx
import { useState } from "react";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { FileBarChart, Download, TrendingUp, Calendar, BarChart3 } from "lucide-react";

export default function SustainabilityReports() {
  const [activeTab, setActiveTab] = useState("monthly");

  const handleDownload = (reportType: string) => {
    console.log(`Downloading ${reportType} report`);
  };

  return (
    <div className="container mx-auto py-6 px-4">
      <div className="mb-6">
        <div className="flex items-center gap-3 mb-2">
          <FileBarChart className="h-8 w-8 text-primary" />
          <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100">Sürdürülebilirlik Raporları</h1>
        </div>
        <p className="text-gray-600 dark:text-gray-400">Sürdürülebilirlik verilerinizi analiz edin ve raporlayın</p>
      </div>

      <Tabs value={activeTab} onValueChange={setActiveTab} className="space-y-6">
        <TabsList className="grid grid-cols-3 w-full max-w-2xl">
          <TabsTrigger value="monthly" data-testid="tab-monthly">
            <Calendar className="h-4 w-4 mr-2" />
            Aylık
          </TabsTrigger>
          <TabsTrigger value="yearly" data-testid="tab-yearly">
            <BarChart3 className="h-4 w-4 mr-2" />
            Yıllık
          </TabsTrigger>
          <TabsTrigger value="custom" data-testid="tab-custom">
            <TrendingUp className="h-4 w-4 mr-2" />
            Özel
          </TabsTrigger>
        </TabsList>

        <TabsContent value="monthly" className="space-y-4">
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            <Card className="hover:shadow-lg transition-shadow">
              <CardHeader>
                <CardTitle className="text-lg">Enerji Tüketimi Raporu</CardTitle>
                <CardDescription>Aylık enerji tüketim özeti</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Rapor Türü:</span>
                    <span className="font-medium dark:text-gray-200">Aylık</span>
                  </div>
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Durum:</span>
                    <span className="text-green-600 dark:text-green-400 font-medium">Hazır</span>
                  </div>
                  <Button 
                    className="w-full" 
                    variant="outline"
                    onClick={() => handleDownload("monthly-energy")}
                    data-testid="button-download-monthly-energy"
                  >
                    <Download className="h-4 w-4 mr-2" />
                    İndir
                  </Button>
                </div>
              </CardContent>
            </Card>

            <Card className="hover:shadow-lg transition-shadow">
              <CardHeader>
                <CardTitle className="text-lg">Su Tüketimi Raporu</CardTitle>
                <CardDescription>Aylık su tüketim özeti</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Rapor Türü:</span>
                    <span className="font-medium dark:text-gray-200">Aylık</span>
                  </div>
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Durum:</span>
                    <span className="text-green-600 dark:text-green-400 font-medium">Hazır</span>
                  </div>
                  <Button 
                    className="w-full" 
                    variant="outline"
                    onClick={() => handleDownload("monthly-water")}
                    data-testid="button-download-monthly-water"
                  >
                    <Download className="h-4 w-4 mr-2" />
                    İndir
                  </Button>
                </div>
              </CardContent>
            </Card>

            <Card className="hover:shadow-lg transition-shadow">
              <CardHeader>
                <CardTitle className="text-lg">Atık Raporu</CardTitle>
                <CardDescription>Aylık atık yönetim özeti</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Rapor Türü:</span>
                    <span className="font-medium dark:text-gray-200">Aylık</span>
                  </div>
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Durum:</span>
                    <span className="text-green-600 dark:text-green-400 font-medium">Hazır</span>
                  </div>
                  <Button 
                    className="w-full" 
                    variant="outline"
                    onClick={() => handleDownload("monthly-waste")}
                    data-testid="button-download-monthly-waste"
                  >
                    <Download className="h-4 w-4 mr-2" />
                    İndir
                  </Button>
                </div>
              </CardContent>
            </Card>
          </div>
        </TabsContent>

        <TabsContent value="yearly" className="space-y-4">
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            <Card className="hover:shadow-lg transition-shadow">
              <CardHeader>
                <CardTitle className="text-lg">Yıllık Enerji Raporu</CardTitle>
                <CardDescription>Yıllık enerji tüketim analizi</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Rapor Türü:</span>
                    <span className="font-medium dark:text-gray-200">Yıllık</span>
                  </div>
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Durum:</span>
                    <span className="text-green-600 dark:text-green-400 font-medium">Hazır</span>
                  </div>
                  <Button 
                    className="w-full" 
                    variant="outline"
                    onClick={() => handleDownload("yearly-energy")}
                    data-testid="button-download-yearly-energy"
                  >
                    <Download className="h-4 w-4 mr-2" />
                    İndir
                  </Button>
                </div>
              </CardContent>
            </Card>

            <Card className="hover:shadow-lg transition-shadow">
              <CardHeader>
                <CardTitle className="text-lg">Yıllık Su Raporu</CardTitle>
                <CardDescription>Yıllık su tüketim analizi</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Rapor Türü:</span>
                    <span className="font-medium dark:text-gray-200">Yıllık</span>
                  </div>
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Durum:</span>
                    <span className="text-green-600 dark:text-green-400 font-medium">Hazır</span>
                  </div>
                  <Button 
                    className="w-full" 
                    variant="outline"
                    onClick={() => handleDownload("yearly-water")}
                    data-testid="button-download-yearly-water"
                  >
                    <Download className="h-4 w-4 mr-2" />
                    İndir
                  </Button>
                </div>
              </CardContent>
            </Card>

            <Card className="hover:shadow-lg transition-shadow">
              <CardHeader>
                <CardTitle className="text-lg">Yıllık Atık Raporu</CardTitle>
                <CardDescription>Yıllık atık yönetim analizi</CardDescription>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Rapor Türü:</span>
                    <span className="font-medium dark:text-gray-200">Yıllık</span>
                  </div>
                  <div className="flex justify-between items-center">
                    <span className="text-sm text-gray-600 dark:text-gray-400">Durum:</span>
                    <span className="text-green-600 dark:text-green-400 font-medium">Hazır</span>
                  </div>
                  <Button 
                    className="w-full" 
                    variant="outline"
                    onClick={() => handleDownload("yearly-waste")}
                    data-testid="button-download-yearly-waste"
                  >
                    <Download className="h-4 w-4 mr-2" />
                    İndir
                  </Button>
                </div>
              </CardContent>
            </Card>
          </div>
        </TabsContent>

        <TabsContent value="custom" className="space-y-4">
          <Card>
            <CardHeader>
              <CardTitle>Özel Rapor Oluştur</CardTitle>
              <CardDescription>Kendi özel raporunuzu oluşturun</CardDescription>
            </CardHeader>
            <CardContent>
              <div className="text-center py-12">
                <TrendingUp className="h-16 w-16 text-gray-400 mx-auto mb-4" />
                <p className="text-gray-600 dark:text-gray-400 mb-4">Özel rapor oluşturma özelliği yakında eklenecek</p>
                <Button variant="outline" disabled data-testid="button-create-custom-report">
                  Yakında
                </Button>
              </div>
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

### 6.8 sustainability-water-footprint.tsx

```tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Activity } from "lucide-react";

export default function SustainabilityWaterFootprint() {
  return (
    <div className="container mx-auto p-6">
      <div className="mb-6">
        <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100 flex items-center gap-2">
          <Activity className="w-8 h-8 text-primary" />
          Su Ayak İzi
        </h1>
        <p className="text-gray-600 dark:text-gray-400 mt-2">
          Su tüketimi ve su ayak izi hesaplamaları
        </p>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Su Ayak İzi Modülü</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-gray-600 dark:text-gray-400">
            Bu bölümde su tüketimi verileri ve su ayak izi hesaplamaları yapılacak.
          </p>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## 7. YARDIMCI KÜTÜPHANELER

### client/src/lib/numberFormat.ts

```typescript
/**
 * Turkish Number Formatting Utilities
 * 
 * Turkish format: 1.234,56 (thousands separator: dot, decimal separator: comma)
 * Standard format: 1234.56 (no thousands separator, decimal separator: dot)
 */

/**
 * Format a number to Turkish format (1.234,56)
 * @param value - The number to format (as string or number)
 * @param decimals - Number of decimal places (default: 2)
 * @returns Formatted string in Turkish format
 */
export function formatTurkishNumber(value: string | number, decimals: number = 2): string {
  if (!value && value !== 0) return '';
  
  // Parse the value to a number
  let numValue: number;
  if (typeof value === 'string') {
    // Check if it's already in Turkish format (contains comma)
    if (value.includes(',')) {
      numValue = parseTurkishNumber(value);
    } else {
      // Standard format from database (e.g., "1613.63")
      numValue = parseFloat(value);
    }
  } else {
    numValue = value;
  }
  
  if (isNaN(numValue)) return '';
  
  // Split into integer and decimal parts
  const parts = numValue.toFixed(decimals).split('.');
  const integerPart = parts[0];
  const decimalPart = parts[1];
  
  // Add thousands separators (dots)
  const formattedInteger = integerPart.replace(/\B(?=(\d{3})+(?!\d))/g, '.');
  
  // Combine with comma as decimal separator
  return decimalPart ? `${formattedInteger},${decimalPart}` : formattedInteger;
}

/**
 * Parse a Turkish formatted number to standard JavaScript number
 * @param value - The Turkish formatted string (1.234,56)
 * @returns Standard number (1234.56)
 */
export function parseTurkishNumber(value: string): number {
  if (!value) return 0;
  
  // Remove all dots (thousands separators)
  // Replace comma with dot (decimal separator)
  const standardFormat = value
    .replace(/\./g, '')  // Remove dots
    .replace(/,/g, '.');  // Replace comma with dot
  
  return parseFloat(standardFormat) || 0;
}

/**
 * Handle keyboard input for Turkish number format
 * Allows: digits, comma (as decimal), backspace, arrow keys
 * Ignores: dot (.)
 * @param event - Keyboard event
 * @param currentValue - Current input value
 * @returns Whether to allow the key press
 */
export function handleTurkishNumberKeyPress(
  event: React.KeyboardEvent<HTMLInputElement>,
  currentValue: string
): boolean {
  const key = event.key;
  
  // Allow control keys
  if (['Backspace', 'Delete', 'Tab', 'Escape', 'Enter', 'ArrowLeft', 'ArrowRight', 'Home', 'End'].includes(key)) {
    return true;
  }
  
  // Allow Ctrl/Cmd combinations
  if (event.ctrlKey || event.metaKey) {
    return true;
  }
  
  // Ignore dot (.)
  if (key === '.') {
    event.preventDefault();
    return false;
  }
  
  // Allow comma only if not already present
  if (key === ',') {
    if (currentValue.includes(',')) {
      event.preventDefault();
      return false;
    }
    return true;
  }
  
  // Allow only digits
  if (!/^\d$/.test(key)) {
    event.preventDefault();
    return false;
  }
  
  return true;
}

/**
 * Format input value as user types (add thousands separators)
 * @param value - Current input value
 * @returns Formatted value with thousands separators
 */
export function formatAsYouType(value: string): string {
  if (!value) return '';
  
  // Split by comma to separate integer and decimal parts
  const parts = value.split(',');
  const integerPart = parts[0].replace(/\./g, ''); // Remove existing dots
  const decimalPart = parts[1] || '';
  
  // Add thousands separators to integer part
  const formattedInteger = integerPart.replace(/\B(?=(\d{3})+(?!\d))/g, '.');
  
  // Reconstruct the value
  return decimalPart ? `${formattedInteger},${decimalPart}` : formattedInteger;
}

/**
 * Calculate emission based on amount and emission factor
 * @param amount - Amount in Turkish format or number
 * @param emissionFactor - Emission factor (from database, in standard format with dot as decimal)
 * @returns Calculated CO2 emission
 */
export function calculateEmission(
  amount: string | number,
  emissionFactor: string | number
): number {
  const amountNum = typeof amount === 'string' ? parseTurkishNumber(amount) : amount;
  // Emission factor comes from database in standard format (dot as decimal), so just parse it
  const factorNum = typeof emissionFactor === 'string' ? parseFloat(emissionFactor) : emissionFactor;
  
  return amountNum * factorNum;
}
```

---

## 8. ROUTING YAPILANDIRMASI

### App.tsx - Sustainability Routes

```tsx
// Import sustainability pages
import SustainabilityDashboard from "@/pages/sustainability-dashboard";
import SustainabilityDataEntry from "@/pages/sustainability-data-entry";
import SustainabilityReports from "@/pages/sustainability-reports";
import SustainabilityCategoryDetail from "@/pages/sustainability-category-detail";
import SustainabilityAnalysis from "@/pages/sustainability-analysis";
import SustainabilityAnalytics from "@/pages/sustainability-analytics";
import SustainabilityWaterFootprint from "@/pages/sustainability-water-footprint";
import SustainabilityParameters from "@/pages/sustainability-parameters";

// Add these routes inside your Router/Switch component:
<Route path="/sustainability/dashboard">
  <SustainabilityDashboard />
</Route>
<Route path="/sustainability/analytics">
  <SustainabilityAnalytics />
</Route>
<Route path="/sustainability/data-entry/:scope/:categoryId/:sourceId">
  <SustainabilityCategoryDetail />
</Route>
<Route path="/sustainability/data-entry">
  <SustainabilityDataEntry />
</Route>
<Route path="/sustainability/water-footprint">
  <SustainabilityWaterFootprint />
</Route>
<Route path="/sustainability/reports">
  <SustainabilityReports />
</Route>
<Route path="/sustainability/parameters">
  <SustainabilityParameters />
</Route>
<Route path="/sustainability/analysis">
  <SustainabilityAnalysis />
</Route>
```

---

## 9. KULLANICI ROLLERİ

Sürdürülebilirlik modülü için gerekli roller:

```typescript
// User roles for sustainability module
type UserRole = 
  | 'sustain_admin'    // Full access - can manage all hospitals' data and parameters
  | 'sustain_user'     // Hospital-specific user - can only manage own hospital's data
  | 'central_admin';   // System admin - full access to everything

// Role-based access:
// - sustain_admin: Can view/edit all hospitals, manage emission parameters
// - sustain_user: Can only view/edit own hospital's emission data
// - central_admin: Full system access including sustainability
```

---

## 10. VERİTABANI MİGRASYONU

Drizzle ile veritabanı migration komutu:

```bash
# Generate migration files
npx drizzle-kit generate:pg

# Push schema changes to database
npx drizzle-kit push:pg
```

---

## 11. ÖNEMLİ NOTLAR

### Emission Factor Hesaplama Mantığı

1. **Elektrik Tüketimi**: `CO₂ (kg) = Tüketim (kWh) × Emisyon Faktörü`
2. **Doğalgaz**: `CO₂ (kg) = Tüketim (m³) × Emisyon Faktörü`
3. **Yakıt**: `CO₂ (kg) = Tüketim (litre) × Emisyon Faktörü`
4. **Soğutucu Gazlar**: `CO₂ (kg) = Dolum Miktarı (kg) × GWP`

### Period Format (Elektrik için)

- Format: `YYYY-DD` (örnek: `2024-01` = 2024 yılı 1. dönem)
- Dönemler: 1-12 arası (aylık okuma dönemleri)

### Turkish Number Format

- Binlik ayırıcı: `.` (nokta)
- Ondalık ayırıcı: `,` (virgül)
- Örnek: 1.234,56 = 1234.56

---

## YEDEK DOSYASI SONU

Bu dosya, Sürdürülebilirlik Modülü'nün bağımsız bir projede çalışması için gereken tüm kodu içermektedir.

Toplam içerik:
- 57 emisyon parametresi (Scope 1/2/3)
- Veritabanı şeması (2 tablo)
- Backend API routes (10+ endpoint)
- Storage interface (10+ method)
- Frontend sayfaları (8 sayfa)
- Yardımcı kütüphaneler (numberFormat)
- Routing yapılandırması

