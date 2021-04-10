---
title: "Trading Intro"
date: 2020-04-05T11:04:41+08:00
---

__Note: This article is automatically translated, please turn to the Chinese version for more accurate expression if possible.__

# stock

## Dutch East India Company

In the 17th century, with the development of navigation technology, many European countries, including the Netherlands, began large-scale colonial expansion. Spices in Europe were very scarce at that time, and trading in spices with Asia could earn dozens of times more profit. Stimulated by high profits, various cities in the Netherlands have formed fleets to trade, but their own way of fighting makes it difficult for them to compete with the Spanish and Portuguese fleets. In order to reduce costs and increase competitiveness, these teams united to form the Dutch East India Company. They raise funds by issuing stocks. Before they travel, everyone can pay to buy the company’s stock. After the company’s trade makes profits, they distribute dividends to those who hold the stock. The world’s first joint-stock company was born. . Through this method of financing the entire society, the East India Company succeeded in turning socially dispersed wealth into its own capital for external expansion.

![1](dutch-east-indian-company.png)

<div class="caption">荷兰东印度公司的标志与船只出海的油画</div>

![1](first-stock.jpg)

<div class="caption">世界上第一支股票。 左侧是股票契约，右侧为分红记录。</div>

The market where stocks are issued by companies and purchased directly by people like this is called the primary market.

## Amsterdam Exchange

The dividends of the Dutch East India Company will not be issued until ten years later. Many people want to realize their stocks before then, and some want to hold more stocks. In this way, a market for buying and selling existing stocks has appeared. , Is called the secondary market. With the development of the secondary market, the local Amsterdam Exchange was established. The exchange centrally records the ownership status of all owners, and the transactions are uniformly matched by the exchange, becoming a credible third party between stock traders and improving the efficiency of stock trading.

![1](amsterdam-stock-exchange.jpg)

<div class="caption">阿姆斯特丹股票交易所</div>

Stock represents the ownership of a company and is a valuable security. When people trade stocks, they are trading company ownership. Therefore, the value of stocks is linked to the value of the company. Speculative capital has also appeared in such a market, that is, buying when the value of the stock is undervalued, and selling when it is overvalued, to earn intermediate plug-ins. The speculation here is not a derogatory term. Appropriate speculative capital will help the market set prices.

## Stock index

In 1884, American journalist Charles Henry Dow selected eleven stocks at that time, weighted the price and the number of stocks, and released the world's first stock index "Dow Jones Transportation Index." Later, he expanded the scope of stocks and released the Dow Jones Industrial Index, which, like the Standard & Poor's Index, became the overall barometer of the stock market.

# futures

## Commodity Futures

The world’s earliest stocks began in the Netherlands, and the world’s earliest futures exchange was not born in the West, but in the Dojima Rice Club in the Edo period of Japan. Dojima is a small town located on the outskirts of Osaka. It has developed land and water transportation and was a rice distribution center in Japan at that time. The nature of the rice crop determines that its price fluctuates with the changes in the sowing and harvesting seasons and weather. During the spring famine season, prices soared; during the harvest season, when farmers gathered at the wharf with a boat full of rice, the price plummeted. In this way, some insightful merchants try to make an appointment with farmers for the agreed price, quantity, and deposit in order to ensure the supply of goods, reduce storage costs, and expand the transaction.

A futures contract includes the quality, quantity, price and delivery time of the commodity. The two parties who signed the contract agreed that at a certain point in the future, the agreed number of products will be traded at the current price, and a certain amount of deposit will be paid in advance.

In order to obtain funds as soon as possible, reduce price risk, and reduce transportation costs, farmers are more willing to make such an appointment transaction.

![1](rice-future-exchange.jpg)

<div class="caption">堂岛大米会所</div>

Like the stock exchange, the Dojima Rice Club has evolved from a trading hub to a futures contract exchange, which manages the trading of contracts and the identification and delivery of goods in a unified manner. The clubhouse standardizes such futures contracts and formulates the unit (package) of the transaction, and the quantity of all transactions is an integer multiple of the unit. At that time, futures trading was unprecedentedly prosperous. There were as many as a thousand market makers, and the contracts they made were nearly forty times more than the actual rice output.

## Financial futures

We call the assets that need to be delivered when the contract expires**Subject**With the development of the concept of futures, the subject matter is no longer limited to various commodities, but can also be financial assets such as stock indexes, currencies, and interest rates.

# Exchange

Although financial products such as stocks and futures serve different purposes, as far as the secondary market is concerned, they are all targets of transactions. We collectively refer to contracts such as stocks and futures as securities, and such exchanges as stock exchanges. As the third party of the buyer and seller, the most basic function of the exchange is to match the transaction. Both buyers and sellers place their orders on the exchange. When a buyer's bid is equal to or higher than the existing seller, the transaction is matched. Transaction and vice versa.

The original exchange maintained an order book for each stock. All sell and buy orders were recorded on it, and orders were matched based on the ledger. With the increase in transaction volume, the enhancement of computer computing power and the popularization of the Internet, exchanges began to use computer systems to automate this process, and people began to trade through the Internet.

## Transaction Entrusted Ledger

Each order is a trading intention, for example, I am willing to sell 100 shares at a price of 30 yuan. The exchange aggregates such orders to form the following transaction entrusted ledger:

![1](limit-order-book.png)

<div class="caption">交易委托账本</div>

As shown in the figure above, the usual transaction prices are discretized. In the figure above, the unit is 0.01 yuan, and a smallest unit is called one.**jump**. At each price, there is a list of orders for that price. Each block in the above figure is an order, and the number inside represents the volume of the order to be traded. For example, the first order on the left indicates that the seller is willing to buy 300 units of securities at a price of 20.07. The lowest price of a sell order is called the selling price, the total amount of the order at this price is called the selling volume, and the following sell orders and buy orders are deduced by analogy.

## Trading Rules

At any moment, the price of a buy order (bid) and a sell order (ask) will not overlap (the overlapped order should be executed immediately). There are usually two types of orders that traders can place: limit orders and market orders. Other types of orders are provided as extended services, but these two are the main ones. A buy limit order means that this order can only be traded at the order price and below, and a sell order is at the order price and above. For example, someone issues a limit buy order with a limit price of 10 yuan, with a quantity of 100, which means that he wants to buy 100 units of securities at a price less than or equal to 10 yuan. Limit orders that are not executed immediately will be recorded on the transaction commission account book. The market order requires immediate execution, so the market order consumes the limit order on the ledger and is executed at the price of the limit order.

![1](order-dynamics.png)

<div class="caption">账本的动态变化</div>

The above figure shows the dynamic change process of a transaction entrusted ledger, and figure 1 is the initial state. In Figure 2, there is a large-volume sell order at the market price, which will be traded with the current best buy order and cause the best buy price to move down. This reflects that when the seller's demand increases, the market price will move downward. Two limit buy orders were added at the moments in Figure 3 and Figure 4. The buy order price in Figure 4 is higher than all the buy orders, which has become the new optimal buying price.

At the same price, there are usually many orders with the same price. When a market order or a limit order that can be traded arrives, the order on the entrusted ledger will be traded in accordance with the rules of the exchange. The rule of most exchanges is FIFO (First In First Out), that is, orders that arrive first at the same price on the ledger have a higher transaction priority.

# Quantitative trading

Where there is a market, there will always be speculators who use events or spatial price inconsistencies to obtain profits. For example, in the seventeenth century, merchants took advantage of the fact that the price of spices differed greatly between Europe and Asia. They bought from Asia and then sold them in Europe to obtain profits. They used spatial price inconsistency to make profits. Another example is that stockholders feel that a certain company has development prospects and buy their stocks in order to sell them after the company's development stocks rise in the future, which is a spatial profit. The essence of speculation is to buy low and sell high to earn the difference between buying and selling.

The technology of speculators has also progressed with the progress of the times. When Dojima Rice Club was prosperous, the trader Souhisa Honma was born. He has studied market fluctuations, seasons, famines, and trader psychology. He also invented a bar chart, which he obtained through his accurate judgment on the price of rice. A lot of wealth. With the electronicization of trading, speculators have also begun to compile their trading strategies into computer programs, and the computer will make trading decisions and place orders. This is the beginning of quantitative trading.

Now, quantitative trading generally refers to the trading process that uses computers and mathematical models to make investment portfolio selection, trading decision generation, order execution and other behaviors through rational analysis and testing.

Quantitative trading has many advantages. First, computer programs are not affected by mentality like humans. For example, most people have the mentality of a gambler, expecting that the price will rise at the next moment during a loss period, but the computer can strictly implement a predetermined stop loss strategy. Secondly, computers have information processing capabilities far beyond that of humans. In the early days, the update speed of market information was slow, and the human eye could still keep up, but now the update frequency of market information is very high, and only computers can fully process this information. The advantages of quantitative trading also include accuracy, objectivity, etc., which are not listed here.

<style>
.caption {
  text-align: center;
  color: gray;
}
</style>
