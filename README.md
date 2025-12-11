import javax.swing.*;
import javax.swing.border.TitledBorder;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.util.*;

public class StockMarketSimulator extends JFrame {

    private JTable marketTable, historyTable, portfolioTable;
    private DefaultTableModel marketModel, historyModel, portfolioModel;
    private JComboBox<String> stockSelector;
    private JTextField amountField;
    private JLabel balanceLabel, notificationLabel, totalProfitLabel;

    private JPanel historyPanel;
    private JButton toggleHistoryBtn;

    private JFrame graphFrame;
    private Map<String, java.util.List<Double>> priceHistory = new HashMap<>();

    private Map<String, Double> stocks;
    private Map<String, Integer> portfolio;
    private Map<String, Double> avgBuyPrice;

    private double balance = 1_00_00_000.00;
    private double totalProfit = 0.00;

    private javax.swing.Timer marketTimer;

    // ---------------- GRAPH PANEL ----------------
    class GraphPanel extends JPanel {
        private String stock;

        public GraphPanel(String stock) {
            this.stock = stock;
            setPreferredSize(new Dimension(600, 400));
        }

        @Override
        protected void paintComponent(Graphics g) {
            super.paintComponent(g);

            java.util.List<Double> history = priceHistory.get(stock);
            if (history == null || history.size() < 2) return;

            Graphics2D g2 = (Graphics2D) g;
            g2.setStroke(new BasicStroke(3));
            g2.setColor(new Color(75, 120, 250));

            int w = getWidth();
            int h = getHeight();
            int margin = 40;

            double max = Collections.max(history);
            double min = Collections.min(history);

            int prevX = margin;
            int prevY = map(history.get(0), min, max, h - margin, margin);

            for (int i = 1; i < history.size(); i++) {
                int x = margin + (i * (w - 2 * margin)) / (history.size() - 1);
                int y = map(history.get(i), min, max, h - margin, margin);

                g2.drawLine(prevX, prevY, x, y);

                prevX = x;
                prevY = y;
            }
        }

        private int map(double value, double min, double max, int newMin, int newMax) {
            if (max == min) return (newMin + newMax) / 2;
            return (int) ((value - min) / (max - min) * (newMin - newMax) + newMax);
        }
    }

    // ---------------------------------------------------

    public StockMarketSimulator() {
        setTitle("📈 Stock Market Simulator");
        setSize(1200, 650);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        getContentPane().setBackground(new Color(235, 240, 255));
        setLayout(new BorderLayout(10, 10));

        JPanel mainPanel = new JPanel(new GridLayout(1, 3, 15, 0));
        mainPanel.setBackground(new Color(235, 240, 255));
        add(mainPanel, BorderLayout.CENTER);

        // ---------------- STOCKS ----------------
        stocks = new LinkedHashMap<>(); // Linked to keep stable ordering

        stocks.put("TCS", 3200.00);
        stocks.put("INFY", 1500.00);
        stocks.put("RELIANCE", 2900.00);
        stocks.put("HDFC", 1650.00);
        stocks.put("SBIN", 780.00);

        stocks.put("WIPRO", 420.00);
        stocks.put("ADANI ENTERPRISES", 2900.00);
        stocks.put("ADANI PORTS", 1250.00);
        stocks.put("ICICI BANK", 980.00);
        stocks.put("AXIS BANK", 1120.00);
        stocks.put("JSW STEEL", 850.00);
        stocks.put("TATA MOTORS", 640.00);
        stocks.put("ONGC", 195.00);
        stocks.put("BHARTI AIRTEL", 950.00);
        stocks.put("HINDUSTAN UNILEVER", 2550.00);

        portfolio = new HashMap<>();
        avgBuyPrice = new HashMap<>();

        // ---------------- LEFT PANEL ----------------
        JPanel tradePanel = createRoundedPanel();
        tradePanel.setLayout(new GridLayout(10, 1, 12, 10));
        tradePanel.setBorder(new TitledBorder("Buy / Sell"));

        stockSelector = new JComboBox<>(stocks.keySet().toArray(new String[0]));
        styleComboBox(stockSelector);

        amountField = new JTextField();
        styleTextField(amountField);

        balanceLabel = new JLabel("Balance: ₹" + String.format("%.2f", balance));
        totalProfitLabel = new JLabel("Total Profit: ₹" + String.format("%.2f", totalProfit));
        notificationLabel = new JLabel("Welcome to Stock Market Simulator!");
        notificationLabel.setForeground(new Color(0, 100, 200));
        notificationLabel.setFont(new Font("Segoe UI", Font.BOLD, 14));

        JButton buyButton = createModernButton("Buy");
        JButton sellButton = createModernButton("Sell");

        buyButton.addActionListener(e -> buyStock());
        sellButton.addActionListener(e -> sellStock());

        tradePanel.add(new JLabel("Select Stock:"));
        tradePanel.add(stockSelector);
        tradePanel.add(new JLabel("Enter Quantity:"));
        tradePanel.add(amountField);
        tradePanel.add(balanceLabel);
        tradePanel.add(totalProfitLabel);
        tradePanel.add(buyButton);
        tradePanel.add(sellButton);
        tradePanel.add(notificationLabel);

        // ---------------- MARKET PANEL ----------------
        JPanel marketPanel = createRoundedPanel();
        marketPanel.setLayout(new BorderLayout());
        marketPanel.setBorder(new TitledBorder("Market Prices"));

        String[] marketCols = {"Stock", "Price (₹)"};
        marketModel = new DefaultTableModel(marketCols, 0) {
            public boolean isCellEditable(int row, int column) { return false; }
        };
        marketTable = new JTable(marketModel);
        formatTable(marketTable);
        updateMarketTable();

        marketTable.addMouseListener(new java.awt.event.MouseAdapter() {
            public void mouseClicked(java.awt.event.MouseEvent evt) {
                int row = marketTable.getSelectedRow();
                if (row == -1) return;

                String stock = (String) marketTable.getValueAt(row, 0);
                openGraph(stock);
            }
        });

        marketPanel.add(new JScrollPane(marketTable), BorderLayout.CENTER);

        // ---------------- RIGHT PANEL ----------------
        JPanel rightPanel = createRoundedPanel();
        rightPanel.setLayout(new BorderLayout());
        rightPanel.setBorder(new TitledBorder("Portfolio & Trade History"));

        // Portfolio Panel (Option A - Table)
        JPanel portfolioPanel = new JPanel(new BorderLayout());
        portfolioPanel.setBorder(new TitledBorder("Your Portfolio"));
        String[] portfolioCols = {"Stock", "Qty", "Avg Buy (₹)", "Current Price (₹)", "P/L (₹)"};
        portfolioModel = new DefaultTableModel(portfolioCols, 0) {
            public boolean isCellEditable(int row, int column) { return false; }
        };
        portfolioTable = new JTable(portfolioModel);
        formatTable(portfolioTable);
        portfolioPanel.add(new JScrollPane(portfolioTable), BorderLayout.CENTER);

        // History Panel (initially hidden)
        toggleHistoryBtn = createModernButton("Show History");
        historyPanel = new JPanel(new BorderLayout());
        historyPanel.setVisible(false);

        String[] cols = {"Action", "Stock", "Qty", "Price (₹)", "Total (₹)", "P/L (₹)"};
        historyModel = new DefaultTableModel(cols, 0) {
            public boolean isCellEditable(int row, int column) { return false; }
        };
        historyTable = new JTable(historyModel);
        formatTable(historyTable);

        historyPanel.add(new JScrollPane(historyTable), BorderLayout.CENTER);

        toggleHistoryBtn.addActionListener(e -> {
            boolean visible = historyPanel.isVisible();
            historyPanel.setVisible(!visible);
            toggleHistoryBtn.setText(visible ? "Show History" : "Hide History");
        });

        // Layout: portfolio at top, history center, toggle button at south
        rightPanel.add(portfolioPanel, BorderLayout.NORTH);
        rightPanel.add(historyPanel, BorderLayout.CENTER);
        rightPanel.add(toggleHistoryBtn, BorderLayout.SOUTH);

        // Add panels to main
        mainPanel.add(tradePanel);
        mainPanel.add(marketPanel);
        mainPanel.add(rightPanel);

        // ---------------- LIVE UPDATE TIMER ----------------
        marketTimer = new javax.swing.Timer(2000, e -> updatePrices());
        marketTimer.start();
    }

    // ---------------- Graph Window ----------------
    private void openGraph(String stock) {
        if (graphFrame != null) graphFrame.dispose();

        graphFrame = new JFrame("📊 Price Graph - " + stock);
        graphFrame.setSize(700, 500);
        graphFrame.setLocationRelativeTo(this);

        graphFrame.add(new GraphPanel(stock));
        graphFrame.setVisible(true);
    }

    // ---------------- Helper Methods ----------------
    private JPanel createRoundedPanel() {
        JPanel panel = new JPanel() {
            protected void paintComponent(Graphics g) {
                g.setColor(Color.WHITE);
                g.fillRoundRect(0, 0, getWidth(), getHeight(), 25, 25);
                super.paintComponent(g);
            }
        };
        panel.setOpaque(false);
        panel.setBorder(BorderFactory.createEmptyBorder(15, 15, 15, 15));
        return panel;
    }

    private JButton createModernButton(String text) {
        JButton btn = new JButton(text);
        btn.setFocusPainted(false);
        btn.setFont(new Font("Segoe UI", Font.BOLD, 16));
        btn.setForeground(Color.WHITE);
        btn.setBackground(new Color(75, 120, 250));
        btn.setBorder(BorderFactory.createEmptyBorder(10, 20, 10, 20));

        btn.addMouseListener(new java.awt.event.MouseAdapter() {
            public void mouseEntered(java.awt.event.MouseEvent evt) {
                btn.setBackground(new Color(55, 90, 220));
            }
            public void mouseExited(java.awt.event.MouseEvent evt) {
                btn.setBackground(new Color(75, 120, 250));
            }
        });

        return btn;
    }

    private void styleComboBox(JComboBox<String> combo) {
        combo.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        combo.setBackground(Color.WHITE);
    }

    private void styleTextField(JTextField txt) {
        txt.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        txt.setBorder(BorderFactory.createLineBorder(new Color(120, 120, 120)));
    }

    private void formatTable(JTable table) {
        table.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        table.setRowHeight(28);
        table.getTableHeader().setFont(new Font("Segoe UI", Font.BOLD, 14));
        table.getTableHeader().setBackground(new Color(210, 220, 255));
    }

    // ---------------- Market Update ----------------
    private void updateMarketTable() {
        marketModel.setRowCount(0);
        for (String stock : stocks.keySet()) {
            marketModel.addRow(new Object[]{
                    stock,
                    String.format("%.2f", stocks.get(stock))
            });
        }
    }

    // ---------------- Portfolio Update ----------------
    private void updatePortfolioTable() {
        portfolioModel.setRowCount(0);
        for (String stock : portfolio.keySet()) {
            int qty = portfolio.get(stock);
            if (qty <= 0) continue;
            double avg = avgBuyPrice.getOrDefault(stock, 0.0);
            double current = stocks.getOrDefault(stock, 0.0);
            double pl = (current - avg) * qty;

            portfolioModel.addRow(new Object[]{
                    stock,
                    qty,
                    String.format("%.2f", avg),
                    String.format("%.2f", current),
                    String.format("%.2f", pl)
            });
        }
    }

    // ---------------- Price Randomization + Graph Update ----------------
    private void updatePrices() {
        Random rand = new Random();

        for (String stock : stocks.keySet()) {
            double price = stocks.get(stock);
            price += (rand.nextDouble() - 0.5) * 100;
            price = Math.max(1, price);
            // round to 2 decimals to keep consistency
            price = Math.round(price * 100.0) / 100.0;
            stocks.put(stock, price);

            priceHistory.putIfAbsent(stock, new ArrayList<>());
            priceHistory.get(stock).add(price);

            if (priceHistory.get(stock).size() > 100)
                priceHistory.get(stock).remove(0);
        }

        updateMarketTable();
        updatePortfolioTable();

        if (graphFrame != null && graphFrame.isVisible()) {
            graphFrame.repaint();
        }
    }

    // ---------------- Buy / Sell Logic ----------------
    private void buyStock() {
        String stock = (String) stockSelector.getSelectedItem();

        try {
            int qty = Integer.parseInt(amountField.getText().trim());
            if (qty <= 0) throw new Exception();

            double price = stocks.get(stock);
            double totalCost = qty * price;
            totalCost = Math.round(totalCost * 100.0) / 100.0;

            if (balance >= totalCost) {
                balance -= totalCost;
                balance = Math.round(balance * 100.0) / 100.0;
                balanceLabel.setText("Balance: ₹" + String.format("%.2f", balance));

                int oldQty = portfolio.getOrDefault(stock, 0);
                double oldAvg = avgBuyPrice.getOrDefault(stock, 0.0);
                double newAvg = (oldAvg * oldQty + qty * price) / (oldQty + qty);

                portfolio.put(stock, oldQty + qty);
                avgBuyPrice.put(stock, newAvg);

                historyModel.addRow(new Object[]{
                        "Buy",
                        stock,
                        qty,
                        String.format("%.2f", price),
                        String.format("%.2f", totalCost),
                        "-"
                });

                updatePortfolioTable();
                showNotification("Bought " + qty + " " + stock, new Color(0, 150, 0));

            } else {
                showNotification("Not enough balance!", Color.RED);
            }

        } catch (Exception e) {
            showNotification("Invalid quantity!", Color.ORANGE);
        }
    }

    private void sellStock() {
        String stock = (String) stockSelector.getSelectedItem();

        try {
            int qty = Integer.parseInt(amountField.getText().trim());
            int owned = portfolio.getOrDefault(stock, 0);

            if (qty <= 0 || owned < qty) throw new Exception();

            double price = stocks.get(stock);
            double buyPrice = avgBuyPrice.getOrDefault(stock, 0.0);
            double totalSell = qty * price;
            double profit = (price - buyPrice) * qty;

            totalSell = Math.round(totalSell * 100.0) / 100.0;
            profit = Math.round(profit * 100.0) / 100.0;

            balance += totalSell;
            balance = Math.round(balance * 100.0) / 100.0;
            balanceLabel.setText("Balance: ₹" + String.format("%.2f", balance));

            portfolio.put(stock, owned - qty);
            if (portfolio.get(stock) == 0)
                avgBuyPrice.remove(stock);

            totalProfit += profit;
            totalProfit = Math.round(totalProfit * 100.0) / 100.0;
            totalProfitLabel.setText("Total Profit: ₹" + String.format("%.2f", totalProfit));

            historyModel.addRow(new Object[]{
                    "Sell",
                    stock,
                    qty,
                    String.format("%.2f", price),
                    String.format("%.2f", totalSell),
                    String.format("%.2f", profit)
            });

            updatePortfolioTable();

            showNotification(
                    profit >= 0 ? "Profit ₹" + String.format("%.2f", profit) : "Loss ₹" + String.format("%.2f", profit),
                    profit >= 0 ? new Color(0, 160, 0) : Color.RED
            );

        } catch (Exception e) {
            showNotification("Invalid quantity or not enough shares!", Color.ORANGE);
        }
    }

    private void showNotification(String msg, Color color) {
        notificationLabel.setText(msg);
        notificationLabel.setForeground(color);
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new StockMarketSimulator().setVisible(true));
    }
}
