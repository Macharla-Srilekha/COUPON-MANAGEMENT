To run this project locally in VS Code, follow these steps. Since this is a standard React + Vite application, the setup is straightforward.

Prerequisites
Node.js: Ensure you have Node.js installed (version 18 or higher is recommended). You can download it from nodejs.org.
VS Code: Installed and ready.
Step 1: Set up the Project Folder
Create a new folder on your computer (e.g., coupon-system).
Open this folder in VS Code.
Step 2: Create the Files
You need to reconstruct the file structure. Based on the code I provided, create these files:

Root Files:

package.json: Copy the dependencies and scripts.
vite.config.ts: Copy the configuration.
tsconfig.json: (Standard React/Vite tsconfig).
tailwind.config.ts (or the generic tailwind setup if you are using v3, but this project uses v4 via CSS import).
Client Folder (client/):

Create client/index.html.
Create client/src/App.tsx.
Create client/src/main.tsx (Entry point).
Create client/src/index.css (Tailwind styles).
Create client/src/lib/mock-api.ts (The Logic Engine).
Create client/src/pages/coupon-system.tsx (The UI).
(Note: If you downloaded the project as a ZIP from Replit, just unzip it and open the folder in VS Code).

Step 3: Install Dependencies
Open the Terminal in VS Code (Ctrl + `) and run:

npm install

This will download React, Vite, Tailwind, and all other libraries listed in package.json.

Step 4: Run the Application
In the VS Code Terminal, run the frontend-specific start command:

npm run dev:client

Why dev:client?: The package.json has a script "dev:client": "vite dev --port 5000". This runs only the React frontend, which is exactly what you need since the logic (Coupon Engine) is running entirely in the browser (in-memory).
Step 5: Open in Browser
Once the server starts, the terminal will show a local URL, usually:
http://localhost:5000

Ctrl+Click that link to open your Coupon System in the browser.

Troubleshooting
"Command not found": Make sure you installed Node.js.
"Missing modules": Run npm install again to ensure all packages are downloaded.
**Port                                                                                           
already in use**: If port 5000 is taken, Vite will automatically try 5001. Check the terminal output for the correct URL.

import { z } from 'zod';

// --- Types ---

export type CouponType = 'percent' | 'flat';

export interface Coupon {
  id: string;
  code: string;
  type: CouponType;
  value: number;
  minCartValue: number;
  eligibleSegments: string[]; // empty means all
  description: string;
  isActive: boolean;
}

export interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

export interface UserContext {
  id: string;
  name: string;
  segment: 'regular' | 'vip' | 'new_user';
}

export interface Cart {
  items: CartItem[];
  user: UserContext;
}

export interface CouponResult {
  coupon: Coupon | null;
  discountAmount: number;
  finalTotal: number;
  message: string;
}

// --- Mock Database ---

const INITIAL_COUPONS: Coupon[] = [
  {
    id: '1',
    code: 'WELCOME10',
    type: 'percent',
    value: 10,
    minCartValue: 0,
    eligibleSegments: ['new_user'],
    description: '10% off for new users',
    isActive: true,
  },
  {
    id: '2',
    code: 'SAVE5',
    type: 'flat',
    value: 5,
    minCartValue: 50,
    eligibleSegments: [],
    description: '$5 off orders over $50',
    isActive: true,
  },
  {
    id: '3',
    code: 'VIP20',
    type: 'percent',
    value: 20,
    minCartValue: 100,
    eligibleSegments: ['vip'],
    description: '20% off for VIPs over $100',
    isActive: true,
  },
];

class CouponEngine {
  private coupons: Coupon[] = [...INITIAL_COUPONS];

  // API: Create Coupon
  createCoupon(couponData: Omit<Coupon, 'id' | 'isActive'>): Coupon {
    const newCoupon: Coupon = {
      ...couponData,
      id: Math.random().toString(36).substr(2, 9),
      isActive: true,
    };
    this.coupons.push(newCoupon);
    return newCoupon;
  }

  // API: Get All Coupons
  getCoupons(): Coupon[] {
    return this.coupons;
  }

  // API: Delete Coupon
  deleteCoupon(id: string) {
    this.coupons = this.coupons.filter(c => c.id !== id);
  }

  // API: Find Best Coupon
  findBestCoupon(cart: Cart): CouponResult {
    const subtotal = cart.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    
    // 1. Filter Eligible Coupons
    const eligibleCoupons = this.coupons.filter(coupon => {
      if (!coupon.isActive) return false;
      
      // Check Min Cart Value
      if (subtotal < coupon.minCartValue) return false;
      
      // Check User Segment
      if (coupon.eligibleSegments.length > 0) {
        if (!coupon.eligibleSegments.includes(cart.user.segment)) {
          return false;
        }
      }
      
      return true;
    });

    if (eligibleCoupons.length === 0) {
      return {
        coupon: null,
        discountAmount: 0,
        finalTotal: subtotal,
        message: 'No eligible coupons found for this cart.',
      };
    }

    // 2. Calculate Discount for each
    let bestCoupon: Coupon | null = null;
    let maxDiscount = -1;

    for (const coupon of eligibleCoupons) {
      let discount = 0;
      if (coupon.type === 'flat') {
        discount = coupon.value;
      } else {
        discount = (subtotal * coupon.value) / 100;
      }
      
      // Ensure discount doesn't exceed total
      if (discount > subtotal) discount = subtotal;

      if (discount > maxDiscount) {
        maxDiscount = discount;
        bestCoupon = coupon;
      }
    }

    return {
      coupon: bestCoupon,
      discountAmount: maxDiscount,
      finalTotal: subtotal - maxDiscount,
      message: bestCoupon ? `Applied ${bestCoupon.code}: Saved $${maxDiscount.toFixed(2)}` : 'No savings applied.',
    };
  }
}

// Export Singleton
export const couponService = new CouponEngine();



import React, { useState, useEffect } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { motion, AnimatePresence } from 'framer-motion';
import { 
  Ticket, 
  ShoppingCart, 
  User, 
  Plus, 
  Trash2, 
  Check, 
  Sparkles,
  ShoppingBag,
  TrendingDown,
  Percent,
  DollarSign,
  Search
} from 'lucide-react';
import { Toaster } from '@/components/ui/sonner';
import { toast } from 'sonner';

import { 
  Card, 
  CardContent, 
  CardDescription, 
  CardFooter, 
  CardHeader, 
  CardTitle 
} from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { 
  Select, 
  SelectContent, 
  SelectItem, 
  SelectTrigger, 
  SelectValue 
} from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import { Separator } from '@/components/ui/separator';
import { Switch } from '@/components/ui/switch';

import { couponService, type Coupon, type Cart, type UserContext, type CartItem } from '@/lib/mock-api';

// --- Validation Schema ---
const couponSchema = z.object({
  code: z.string().min(3, 'Code must be at least 3 characters').toUpperCase(),
  type: z.enum(['percent', 'flat']),
  value: z.coerce.number().min(0.01, 'Value must be positive'),
  minCartValue: z.coerce.number().min(0),
  eligibleSegments: z.string().optional(), // We'll handle parsing manually
  description: z.string().min(5, 'Description required'),
});

type CouponFormValues = z.infer<typeof couponSchema>;

// --- Components ---

function CouponCreator({ onUpdate }: { onUpdate: () => void }) {
  const form = useForm<CouponFormValues>({
    resolver: zodResolver(couponSchema),
    defaultValues: {
      code: '',
      type: 'percent',
      value: 10,
      minCartValue: 0,
      description: '',
      eligibleSegments: 'all',
    }
  });

  const onSubmit = (data: CouponFormValues) => {
    const segments = data.eligibleSegments === 'all' ? [] : [data.eligibleSegments!];
    
    couponService.createCoupon({
      code: data.code,
      type: data.type,
      value: data.value,
      minCartValue: data.minCartValue,
      description: data.description,
      eligibleSegments: segments,
    });
    
    toast.success('Coupon created successfully');
    form.reset();
    onUpdate();
  };

  return (
    <Card className="h-full border-0 shadow-none bg-transparent">
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <Plus className="w-5 h-5 text-accent" /> Create New Coupon
        </CardTitle>
        <CardDescription>Define rules and constraints for new discounts.</CardDescription>
      </CardHeader>
      <CardContent>
        <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div className="space-y-2">
              <Label htmlFor="code">Code</Label>
              <div className="relative">
                <Input id="code" placeholder="SUMMER25" className="font-mono uppercase pl-9" {...form.register('code')} />
                <Ticket className="w-4 h-4 absolute left-3 top-3 text-muted-foreground" />
              </div>
              {form.formState.errors.code && <p className="text-xs text-destructive">{form.formState.errors.code.message}</p>}
            </div>
            
            <div className="space-y-2">
              <Label htmlFor="type">Type</Label>
              <Select 
                onValueChange={(val) => form.setValue('type', val as 'percent' | 'flat')}
                defaultValue={form.getValues('type')}
              >
                <SelectTrigger>
                  <SelectValue placeholder="Select type" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="percent">Percentage (%)</SelectItem>
                  <SelectItem value="flat">Flat Amount ($)</SelectItem>
                </SelectContent>
              </Select>
            </div>
          </div>

          <div className="grid grid-cols-2 gap-4">
            <div className="space-y-2">
              <Label htmlFor="value">Discount Value</Label>
              <Input type="number" id="value" {...form.register('value')} />
            </div>
             <div className="space-y-2">
              <Label htmlFor="minCartValue">Min Cart Value ($)</Label>
              <Input type="number" id="minCartValue" {...form.register('minCartValue')} />
            </div>
          </div>

          <div className="space-y-2">
            <Label htmlFor="segment">Eligible User Segment</Label>
            <Select 
              onValueChange={(val) => form.setValue('eligibleSegments', val)}
              defaultValue="all"
            >
              <SelectTrigger>
                <SelectValue placeholder="Select segment" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Users</SelectItem>
                <SelectItem value="vip">VIP Only</SelectItem>
                <SelectItem value="new_user">New Users Only</SelectItem>
                <SelectItem value="regular">Regular Only</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <div className="space-y-2">
            <Label htmlFor="description">Description</Label>
            <Input id="description" placeholder="e.g. 20% off for summer sale" {...form.register('description')} />
            {form.formState.errors.description && <p className="text-xs text-destructive">{form.formState.errors.description.message}</p>}
          </div>

          <Button type="submit" className="w-full bg-primary text-primary-foreground hover:bg-primary/90">
            Create Coupon
          </Button>
        </form>
      </CardContent>
    </Card>
  );
}

function CouponList({ coupons, onDelete }: { coupons: Coupon[], onDelete: () => void }) {
  return (
    <div className="space-y-3">
      <h3 className="text-sm font-medium text-muted-foreground mb-4">Active Coupons ({coupons.length})</h3>
      <AnimatePresence>
        {coupons.map((coupon) => (
          <motion.div
            key={coupon.id}
            initial={{ opacity: 0, y: 10 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, scale: 0.95 }}
            layout
          >
            <Card className="overflow-hidden border border-border/50 hover:border-accent/50 transition-colors group">
              <div className="p-4 flex items-center justify-between">
                <div className="flex items-start gap-4">
                  <div className="h-10 w-10 rounded-full bg-accent/10 flex items-center justify-center text-accent">
                    {coupon.type === 'percent' ? <Percent className="w-5 h-5" /> : <DollarSign className="w-5 h-5" />}
                  </div>
                  <div>
                    <div className="flex items-center gap-2">
                      <span className="font-mono font-bold text-lg">{coupon.code}</span>
                      <Badge variant="secondary" className="text-xs font-normal">
                        {coupon.type === 'percent' ? `${coupon.value}% OFF` : `$${coupon.value} OFF`}
                      </Badge>
                    </div>
                    <p className="text-sm text-muted-foreground">{coupon.description}</p>
                    <div className="flex gap-2 mt-2">
                      {coupon.minCartValue > 0 && (
                        <Badge variant="outline" className="text-[10px] h-5">Min ${coupon.minCartValue}</Badge>
                      )}
                      {coupon.eligibleSegments.length > 0 ? (
                        coupon.eligibleSegments.map(s => <Badge key={s} variant="outline" className="text-[10px] h-5 bg-blue-50 text-blue-700 border-blue-200 capitalize">{s}</Badge>)
                      ) : (
                        <Badge variant="outline" className="text-[10px] h-5 bg-green-50 text-green-700 border-green-200">Everyone</Badge>
                      )}
                    </div>
                  </div>
                </div>
                <Button 
                  variant="ghost" 
                  size="icon" 
                  onClick={() => {
                    couponService.deleteCoupon(coupon.id);
                    onDelete();
                  }}
                  className="text-muted-foreground hover:text-destructive opacity-0 group-hover:opacity-100 transition-opacity"
                >
                  <Trash2 className="w-4 h-4" />
                </Button>
              </div>
            </Card>
          </motion.div>
        ))}
      </AnimatePresence>
      {coupons.length === 0 && (
        <div className="text-center py-12 text-muted-foreground border border-dashed rounded-lg">
          No coupons active
        </div>
      )}
    </div>
  );
}

// --- Cart Simulator ---

function CartSimulator() {
  const [user, setUser] = useState<UserContext>({ id: 'u1', name: 'John Doe', segment: 'regular' });
  const [cartItems, setCartItems] = useState<CartItem[]>([
    { id: 'p1', name: 'Premium Headphones', price: 150, quantity: 1 },
    { id: 'p2', name: 'USB-C Cable', price: 20, quantity: 2 },
  ]);
  const [result, setResult] = useState<any>(null);

  const subtotal = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);

  const checkCoupons = () => {
    const res = couponService.findBestCoupon({ items: cartItems, user });
    setResult(res);
    if (res.coupon) {
      toast.success(`Found deal: ${res.coupon.code}`);
    } else {
      toast.info('No eligible coupons found');
    }
  };

  const addItem = () => {
    const newItem = { id: `p${Date.now()}`, name: 'New Product', price: 50, quantity: 1 };
    setCartItems([...cartItems, newItem]);
    setResult(null); // Reset result on change
  };

  const removeItem = (id: string) => {
    setCartItems(cartItems.filter(i => i.id !== id));
    setResult(null);
  };

  return (
    <div className="grid grid-cols-1 lg:grid-cols-2 gap-8 h-full">
      {/* Left: Cart Configuration */}
      <div className="space-y-6">
        <Card>
          <CardHeader>
            <CardTitle className="flex items-center gap-2">
              <User className="w-5 h-5 text-blue-500" /> User Context
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="space-y-2">
              <Label>Customer Segment</Label>
              <Select 
                value={user.segment} 
                onValueChange={(v) => {
                  setUser({ ...user, segment: v as any });
                  setResult(null);
                }}
              >
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="regular">Regular Customer</SelectItem>
                  <SelectItem value="vip">VIP Member</SelectItem>
                  <SelectItem value="new_user">New User</SelectItem>
                </SelectContent>
              </Select>
              <p className="text-xs text-muted-foreground mt-2">
                Simulates different users logging in. VIPs might see better deals.
              </p>
            </div>
          </CardContent>
        </Card>

        <Card className="flex-1 flex flex-col">
          <CardHeader className="flex flex-row items-center justify-between pb-2">
            <CardTitle className="flex items-center gap-2">
              <ShoppingCart className="w-5 h-5 text-green-500" /> Shopping Cart
            </CardTitle>
            <Button variant="outline" size="sm" onClick={addItem}>
              <Plus className="w-4 h-4 mr-1" /> Add Item
            </Button>
          </CardHeader>
          <CardContent className="flex-1">
            <div className="space-y-3">
              <AnimatePresence>
                {cartItems.map((item) => (
                  <motion.div 
                    key={item.id}
                    initial={{ opacity: 0, height: 0 }}
                    animate={{ opacity: 1, height: 'auto' }}
                    exit={{ opacity: 0, height: 0 }}
                    className="flex items-center justify-between p-3 border rounded-lg bg-background/50"
                  >
                    <div>
                      <div className="font-medium">{item.name}</div>
                      <div className="text-sm text-muted-foreground">${item.price} x {item.quantity}</div>
                    </div>
                    <div className="flex items-center gap-3">
                      <div className="font-mono font-bold">${item.price * item.quantity}</div>
                      <Button variant="ghost" size="icon" className="h-8 w-8 text-muted-foreground hover:text-destructive" onClick={() => removeItem(item.id)}>
                        <Trash2 className="w-4 h-4" />
                      </Button>
                    </div>
                  </motion.div>
                ))}
              </AnimatePresence>
              {cartItems.length === 0 && (
                <div className="text-center py-8 text-muted-foreground">Cart is empty</div>
              )}
            </div>
          </CardContent>
          <CardFooter className="flex-col gap-4 border-t pt-6">
            <div className="flex w-full justify-between items-center text-lg font-medium">
              <span>Subtotal</span>
              <span>${subtotal.toFixed(2)}</span>
            </div>
            <Button 
              size="lg" 
              className="w-full bg-black hover:bg-zinc-800 text-white shadow-lg"
              onClick={checkCoupons}
              disabled={cartItems.length === 0}
            >
              <Search className="w-4 h-4 mr-2" /> Find Best Coupon
            </Button>
          </CardFooter>
        </Card>
      </div>

      {/* Right: Results Panel */}
      <div className="space-y-6">
        <div className="h-full flex flex-col">
          <h2 className="text-lg font-semibold mb-4 flex items-center gap-2">
            <Sparkles className="w-5 h-5 text-accent" /> Optimization Result
          </h2>
          
          <div className="flex-1 border-2 border-dashed border-border rounded-xl p-6 bg-muted/20 flex flex-col items-center justify-center text-center relative overflow-hidden">
            {result ? (
              <motion.div 
                initial={{ scale: 0.9, opacity: 0 }}
                animate={{ scale: 1, opacity: 1 }}
                className="w-full max-w-md relative z-10"
              >
                {result.coupon ? (
                  <div className="bg-white dark:bg-zinc-900 rounded-xl shadow-2xl border border-border overflow-hidden">
                    <div className="bg-accent text-accent-foreground p-6">
                      <div className="uppercase tracking-wider text-xs font-bold opacity-80 mb-1">Best Coupon Found</div>
                      <div className="text-4xl font-mono font-bold tracking-tight">{result.coupon.code}</div>
                    </div>
                    <div className="p-8 space-y-6">
                      <div className="flex justify-between items-center pb-4 border-b">
                        <span className="text-muted-foreground">Original Total</span>
                        <span className="text-xl decoration-slate-400 line-through">${subtotal.toFixed(2)}</span>
                      </div>
                      <div className="flex justify-between items-center pb-4 border-b">
                        <span className="text-green-600 font-medium flex items-center gap-2">
                          <TrendingDown className="w-4 h-4" /> Discount Applied
                        </span>
                        <span className="text-xl text-green-600 font-bold">-${result.discountAmount.toFixed(2)}</span>
                      </div>
                      <div className="flex justify-between items-center pt-2">
                        <span className="font-bold text-lg">Final Total</span>
                        <span className="text-4xl font-bold tracking-tight">${result.finalTotal.toFixed(2)}</span>
                      </div>
                      <div className="bg-blue-50 text-blue-700 p-3 rounded-lg text-sm border border-blue-100">
                        {result.message}
                      </div>
                    </div>
                  </div>
                ) : (
                   <div className="text-muted-foreground space-y-4">
                     <div className="w-16 h-16 rounded-full bg-muted flex items-center justify-center mx-auto">
                        <Ticket className="w-8 h-8 opacity-50" />
                     </div>
                     <p className="text-lg font-medium">No coupons match your criteria.</p>
                     <p className="text-sm max-w-xs mx-auto">Try increasing your cart value or changing the user segment.</p>
                   </div>
                )}
              </motion.div>
            ) : (
              <div className="text-muted-foreground space-y-2">
                <ShoppingBag className="w-12 h-12 mx-auto opacity-20" />
                <p>Add items and click "Find Best Coupon"</p>
              </div>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}

// --- Main Page ---

export default function CouponSystemPage() {
  const [coupons, setCoupons] = useState<Coupon[]>([]);

  const refreshCoupons = () => {
    setCoupons([...couponService.getCoupons()]);
  };

  useEffect(() => {
    refreshCoupons();
  }, []);

  return (
    <div className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b bg-card sticky top-0 z-50">
        <div className="container mx-auto px-6 h-16 flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="w-8 h-8 bg-black text-white rounded-lg flex items-center justify-center font-bold font-mono">
              CP
            </div>
            <h1 className="font-bold text-xl tracking-tight">CouponEngine</h1>
          </div>
          <nav className="flex items-center gap-4">
            <Button variant="ghost" size="sm" className="text-muted-foreground">Documentation</Button>
            <div className="w-px h-4 bg-border"></div>
            <div className="flex items-center gap-2 text-sm text-muted-foreground">
              <span className="w-2 h-2 rounded-full bg-green-500"></span>
              System Operational
            </div>
          </nav>
        </div>
      </header>

      <main className="container mx-auto px-6 py-8">
        <Tabs defaultValue="simulator" className="space-y-8">
          <div className="flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4">
             <div className="space-y-1">
               <h2 className="text-2xl font-semibold tracking-tight">System Dashboard</h2>
               <p className="text-muted-foreground">Manage rules and simulate checkout scenarios.</p>
             </div>
             <TabsList className="bg-muted/50 p-1">
               <TabsTrigger value="manage" className="px-6">Manage Coupons</TabsTrigger>
               <TabsTrigger value="simulator" className="px-6">Checkout Simulator</TabsTrigger>
             </TabsList>
          </div>

          <TabsContent value="manage" className="space-y-6 animate-in fade-in slide-in-from-bottom-4 duration-500">
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
              <div className="lg:col-span-1">
                <CouponCreator onUpdate={refreshCoupons} />
              </div>
              <div className="lg:col-span-2">
                <Card>
                  <CardHeader>
                    <CardTitle>Active Coupons</CardTitle>
                    <CardDescription>Live discount rules currently in the system.</CardDescription>
                  </CardHeader>
                  <CardContent>
                    <CouponList coupons={coupons} onDelete={refreshCoupons} />
                  </CardContent>
                </Card>
              </div>
            </div>
          </TabsContent>

          <TabsContent value="simulator" className="animate-in fade-in slide-in-from-bottom-4 duration-500">
             <CartSimulator />
          </TabsContent>
        </Tabs>
      </main>
    </div>
  );
}
