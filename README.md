<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Perrin Sequence Inventory Tracker</title>
    
    <!-- Tailwind CSS -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tailwindcss/3.4.1/tailwind.min.js"></script>
    
    <!-- React and ReactDOM -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.development.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.development.js"></script>
    
    <!-- Babel for JSX -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.24.0/babel.min.js"></script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f4f4f5;
        }

        /* Custom shadcn-inspired styles */
        .card {
            background-color: white;
            border-radius: 0.5rem;
            box-shadow: 0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1);
        }

        .badge {
            display: inline-flex;
            align-items: center;
            border-radius: 9999px;
            padding: 0.25rem 0.75rem;
            font-size: 0.875rem;
            font-weight: 500;
        }

        .badge-secondary {
            background-color: #f4f4f5;
            color: #18181b;
        }

        .badge-outline {
            border: 1px solid #e4e4e7;
            color: #71717a;
        }

        .alert {
            border-radius: 0.5rem;
            padding: 1rem;
            border: 1px solid #fca5a5;
            background-color: #fee2e2;
            color: #b91c1c;
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        import React, { useState, useEffect } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { RefreshCw, Clock, Percent, Zap, AlertCircle } from 'lucide-react';

const REFRESH_INTERVAL = 300000; // 5 minutes in milliseconds
const API_ENDPOINT = 'https://api.warframestat.us/pc/syndicate';

const PerrinTracker = () => {
  const [inventory, setInventory] = useState([]);
  const [lastUpdated, setLastUpdated] = useState(null);
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [autoRefresh, setAutoRefresh] = useState(true);

  const fetchInventory = async () => {
    try {
      setIsLoading(true);
      const response = await fetch(API_ENDPOINT);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const data = await response.json();
      const perrinData = data.find(syndicate => 
        syndicate.name.toLowerCase().includes('perrin')
      );

      if (!perrinData) {
        throw new Error('Perrin Sequence data not found');
      }

      // Transform API data to match our format
      const transformedInventory = perrinData.offerings.map(item => ({
        name: item.name,
        masteryRank: item.masteryReq || 0,
        basePrice: item.credits,
        syndicatePoints: item.cost,
        bonusType: item.type === 'Weapon' ? 'Sequence' : null,
        bonusEffect: item.type === 'Weapon' ? 'On Proc: Restore 25% Base Energy' : null,
        bonusPercent: 25,
        inStock: true,
        type: determineItemType(item.name)
      }));

      setInventory(transformedInventory);
      setLastUpdated(new Date());
      setError(null);
    } catch (err) {
      setError(err.message);
      console.error('Error fetching inventory:', err);
    } finally {
      setIsLoading(false);
    }
  };

  // Helper function to determine item type
  const determineItemType = (name) => {
    if (name.includes('Secura')) {
      return name.includes('Dual') || name.includes('Akstiletto') ? 'Secondary' : 'Primary';
    }
    return 'Mod';
  };

  // Initial fetch and auto-refresh setup
  useEffect(() => {
    fetchInventory();

    // Set up auto-refresh
    let intervalId;
    if (autoRefresh) {
      intervalId = setInterval(fetchInventory, REFRESH_INTERVAL);
    }

    return () => {
      if (intervalId) {
        clearInterval(intervalId);
      }
    };
  }, [autoRefresh]);

  const handleManualRefresh = () => {
    fetchInventory();
  };

  const toggleAutoRefresh = () => {
    setAutoRefresh(!autoRefresh);
  };

  const formatPrice = (credits) => {
    return credits.toLocaleString() + " Credits";
  };

  const getBonusColor = (bonusType) => {
    const colors = {
      'Sequence': 'text-blue-500',
      'Entropy': 'text-red-500',
    };
    return colors[bonusType] || 'text-gray-500';
  };

  const getTimeUntilNextRefresh = () => {
    if (!autoRefresh || !lastUpdated) return null;
    const nextUpdate = new Date(lastUpdated.getTime() + REFRESH_INTERVAL);
    const now = new Date();
    const diff = nextUpdate - now;
    const minutes = Math.floor(diff / 60000);
    const seconds = Math.floor((diff % 60000) / 1000);
    return `${minutes}m ${seconds}s`;
  };

  return (
    <div className="max-w-4xl mx-auto p-4 space-y-4">
      <Card className="w-full">
        <CardHeader>
          <div className="flex justify-between items-center">
            <div>
              <CardTitle className="text-xl">Ergo Glast - Perrin Sequence</CardTitle>
              <p className="text-sm text-gray-500 mt-1">
                "A fair exchange is no robbery."
              </p>
            </div>
            <div className="flex items-center gap-2">
              <div className="flex flex-col items-end text-sm">
                <div className="flex items-center gap-2">
                  <Clock className="h-4 w-4" />
                  <span>
                    {lastUpdated ? lastUpdated.toLocaleTimeString() : 'Never'}
                  </span>
                </div>
                {autoRefresh && (
                  <span className="text-gray-500">
                    Next update in: {getTimeUntilNextRefresh()}
                  </span>
                )}
              </div>
              <button 
                onClick={handleManualRefresh}
                disabled={isLoading}
                className="p-2 rounded-full hover:bg-gray-100 disabled:opacity-50"
              >
                <RefreshCw className={`h-4 w-4 ${isLoading ? 'animate-spin' : ''}`} />
              </button>
              <button
                onClick={toggleAutoRefresh}
                className={`px-3 py-1 rounded-full text-sm ${
                  autoRefresh ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800'
                }`}
              >
                {autoRefresh ? 'Auto-refresh On' : 'Auto-refresh Off'}
              </button>
            </div>
          </div>
        </CardHeader>
        <CardContent>
          {error && (
            <Alert variant="destructive" className="mb-4">
              <AlertCircle className="h-4 w-4" />
              <AlertDescription>
                Error loading inventory: {error}. Will retry automatically.
              </AlertDescription>
            </Alert>
          )}
          
          {isLoading && inventory.length === 0 ? (
            <div className="text-center py-8">
              <RefreshCw className="h-8 w-8 animate-spin mx-auto mb-4" />
              <p>Loading inventory...</p>
            </div>
          ) : (
            <div className="space-y-4">
              {inventory.map((item, index) => (
                <div key={index} className="border rounded-lg p-4">
                  <div className="flex justify-between items-start mb-2">
                    <div>
                      <h3 className="text-lg font-semibold">{item.name}</h3>
                      <div className="flex gap-2 mt-1">
                        <Badge variant="secondary">
                          {item.type}
                        </Badge>
                        {item.masteryRank > 0 && (
                          <Badge variant="outline">
                            MR {item.masteryRank}
                          </Badge>
                        )}
                      </div>
                    </div>
                    <div className="text-right">
                      <div className="font-medium">{formatPrice(item.basePrice)}</div>
                      <div className="text-sm text-gray-500">
                        {item.syndicatePoints.toLocaleString()} Standing
                      </div>
                    </div>
                  </div>
                  
                  {item.bonusType && (
                    <div className="mt-2 space-y-1">
                      <div className="flex items-center gap-2">
                        <Zap className="h-4 w-4" />
                        <span className={`font-medium ${getBonusColor(item.bonusType)}`}>
                          {item.bonusType}
                        </span>
                      </div>
                      <div className="flex items-center gap-2 text-sm text-gray-600">
                        <Percent className="h-4 w-4" />
                        {item.bonusEffect}
                      </div>
                    </div>
                  )}
                </div>
              ))}
            </div>
          )}
        </CardContent>
      </Card>
    </div>
  );
};

export default PerrinTracker;
        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<PerrinTracker />);
    </script>
</body>
</html>
