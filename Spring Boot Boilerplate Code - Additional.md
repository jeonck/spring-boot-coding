# Spring Boot Boilerplate Code - 추가 구성 요소

> 기본 보일러플레이트에서 누락된 나머지 Service, Controller 및 완전한 테스트 코드

## 📦 나머지 Service 클래스들

### ProductService

```java
// service/ProductService.java
package com.example.app.service;

import com.example.app.dto.request.CreateProductRequest;
import com.example.app.dto.request.UpdateProductRequest;
import com.example.app.dto.response.ProductResponse;
import com.example.app.dto.response.PageResponse;
import com.example.app.entity.Product;
import com.example.app.entity.ProductCategory;
import com.example.app.entity.ProductStatus;
import com.example.app.exception.DuplicateResourceException;
import com.example.app.exception.ResourceNotFoundException;
import com.example.app.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class ProductService {
    
    private final ProductRepository productRepository;
    
    @Cacheable(value = "products", key = "#pageable.pageNumber + '_' + #pageable.pageSize")
    public PageResponse<ProductResponse> getAllProducts(Pageable pageable) {
        Page<Product> products = productRepository.findAll(pageable);
        Page<ProductResponse> productResponses = products.map(this::convertToResponse);
        return PageResponse.of(productResponses);
    }
    
    @Cacheable(value = "product", key = "#id")
    public ProductResponse getProductById(Long id) {
        Product product = findProductById(id);
        return convertToResponse(product);
    }
    
    @Cacheable(value = "product", key = "#sku")
    public ProductResponse getProductBySku(String sku) {
        Product product = productRepository.findBySku(sku)
            .orElseThrow(() -> new ResourceNotFoundException("상품을 찾을 수 없습니다: " + sku));
        return convertToResponse(product);
    }
    
    public PageResponse<ProductResponse> getProductsByCategory(ProductCategory category, Pageable pageable) {
        Page<Product> products = productRepository.findByCategoryAndStatus(category, ProductStatus.ACTIVE, pageable);
        Page<ProductResponse> productResponses = products.map(this::convertToResponse);
        return PageResponse.of(productResponses);
    }
    
    public PageResponse<ProductResponse> searchProducts(String keyword, Pageable pageable) {
        Page<Product> products = productRepository.findByKeyword(keyword, pageable);
        Page<ProductResponse> productResponses = products.map(this::convertToResponse);
        return PageResponse.of(productResponses);
    }
    
    public PageResponse<ProductResponse> getProductsByPriceRange(BigDecimal minPrice, BigDecimal maxPrice, Pageable pageable) {
        Page<Product> products = productRepository.findByPriceRange(minPrice, maxPrice, pageable);
        Page<ProductResponse> productResponses = products.map(this::convertToResponse);
        return PageResponse.of(productResponses);
    }
    
    public List<ProductResponse> getLowStockProducts(Integer threshold) {
        List<Product> products = productRepository.findLowStockProducts(threshold);
        return products.stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }
    
    @Transactional
    @CacheEvict(value = {"products", "product"}, allEntries = true)
    public ProductResponse createProduct(CreateProductRequest request) {
        validateProductConstraints(request);
        
        Product product = Product.builder()
            .name(request.getName())
            .description(request.getDescription())
            .price(request.getPrice())
            .stockQuantity(request.getStockQuantity())
            .sku(request.getSku())
            .category(request.getCategory())
            .status(ProductStatus.ACTIVE)
            .imageUrl(request.getImageUrl())
            .weight(request.getWeight())
            .dimensions(request.getDimensions())
            .build();
        
        Product savedProduct = productRepository.save(product);
        log.info("상품이 생성되었습니다: {} (SKU: {})", savedProduct.getName(), savedProduct.getSku());
        
        return convertToResponse(savedProduct);
    }
    
    @Transactional
    @CacheEvict(value = {"products", "product"}, allEntries = true)
    public ProductResponse updateProduct(Long id, UpdateProductRequest request) {
        Product product = findProductById(id);
        
        // SKU가 변경되는 경우 중복 확인
        if (request.getSku() != null && !product.getSku().equals(request.getSku())) {
            if (productRepository.findBySku(request.getSku()).isPresent()) {
                throw new DuplicateResourceException("이미 사용 중인 SKU입니다: " + request.getSku());
            }
            product.setSku(request.getSku());
        }
        
        updateProductFields(product, request);
        
        Product updatedProduct = productRepository.save(product);
        log.info("상품 정보가 업데이트되었습니다: {}", updatedProduct.getName());
        
        return convertToResponse(updatedProduct);
    }
    
    @Transactional
    @CacheEvict(value = {"products", "product"}, allEntries = true)
    public void deleteProduct(Long id) {
        Product product = findProductById(id);
        product.setStatus(ProductStatus.DISCONTINUED);
        productRepository.save(product);
        log.info("상품이 삭제되었습니다: {}", product.getName());
    }
    
    @Transactional
    public void updateStock(Long productId, int quantity) {
        Product product = findProductById(productId);
        
        if (quantity > 0) {
            product.increaseStock(quantity);
        } else {
            product.decreaseStock(Math.abs(quantity));
        }
        
        productRepository.save(product);
        log.info("상품 재고가 업데이트되었습니다: {} (변경량: {})", product.getName(), quantity);
    }
    
    private Product findProductById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("상품을 찾을 수 없습니다: " + id));
    }
    
    private void validateProductConstraints(CreateProductRequest request) {
        if (request.getSku() != null && productRepository.findBySku(request.getSku()).isPresent()) {
            throw new DuplicateResourceException("이미 사용 중인 SKU입니다: " + request.getSku());
        }
    }
    
    private void updateProductFields(Product product, UpdateProductRequest request) {
        if (request.getName() != null) {
            product.setName(request.getName());
        }
        if (request.getDescription() != null) {
            product.setDescription(request.getDescription());
        }
        if (request.getPrice() != null) {
            product.setPrice(request.getPrice());
        }
        if (request.getStockQuantity() != null) {
            product.setStockQuantity(request.getStockQuantity());
        }
        if (request.getCategory() != null) {
            product.setCategory(request.getCategory());
        }
        if (request.getImageUrl() != null) {
            product.setImageUrl(request.getImageUrl());
        }
        if (request.getWeight() != null) {
            product.setWeight(request.getWeight());
        }
        if (request.getDimensions() != null) {
            product.setDimensions(request.getDimensions());
        }
    }
    
    private ProductResponse convertToResponse(Product product) {
        return ProductResponse.builder()
            .id(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice())
            .stockQuantity(product.getStockQuantity())
            .sku(product.getSku())
            .category(product.getCategory())
            .status(product.getStatus())
            .imageUrl(product.getImageUrl())
            .weight(product.getWeight())
            .dimensions(product.getDimensions())
            .available(product.isAvailable())
            .createdAt(product.getCreatedAt())
            .updatedAt(product.getUpdatedAt())
            .build();
    }
}
```

### OrderService

```java
// service/OrderService.java
package com.example.app.service;

import com.example.app.dto.request.CreateOrderRequest;
import com.example.app.dto.request.OrderItemRequest;
import com.example.app.dto.response.OrderResponse;
import com.example.app.dto.response.OrderItemResponse;
import com.example.app.dto.response.PageResponse;
import com.example.app.entity.*;
import com.example.app.exception.BusinessException;
import com.example.app.exception.ResourceNotFoundException;
import com.example.app.repository.OrderRepository;
import com.example.app.repository.ProductRepository;
import com.example.app.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    
    public PageResponse<OrderResponse> getAllOrders(Pageable pageable) {
        Page<Order> orders = orderRepository.findAll(pageable);
        Page<OrderResponse> orderResponses = orders.map(this::convertToResponse);
        return PageResponse.of(orderResponses);
    }
    
    public OrderResponse getOrderById(Long id) {
        Order order = findOrderById(id);
        return convertToResponse(order);
    }
    
    public OrderResponse getOrderByOrderNumber(String orderNumber) {
        Order order = orderRepository.findByOrderNumber(orderNumber)
            .orElseThrow(() -> new ResourceNotFoundException("주문을 찾을 수 없습니다: " + orderNumber));
        return convertToResponse(order);
    }
    
    public PageResponse<OrderResponse> getOrdersByUser(Long userId, Pageable pageable) {
        Page<Order> orders = orderRepository.findByUserId(userId, pageable);
        Page<OrderResponse> orderResponses = orders.map(this::convertToResponse);
        return PageResponse.of(orderResponses);
    }
    
    public PageResponse<OrderResponse> getOrdersByStatus(OrderStatus status, Pageable pageable) {
        Page<Order> orders = orderRepository.findByStatus(status, pageable);
        Page<OrderResponse> orderResponses = orders.map(this::convertToResponse);
        return PageResponse.of(orderResponses);
    }
    
    @Transactional
    public OrderResponse createOrder(Long userId, CreateOrderRequest request) {
        User user = findUserById(userId);
        
        Order order = Order.builder()
            .orderNumber(generateOrderNumber())
            .user(user)
            .status(OrderStatus.PENDING)
            .totalAmount(BigDecimal.ZERO)
            .shippingAddress(request.getShippingAddress())
            .billingAddress(request.getBillingAddress())
            .paymentMethod(request.getPaymentMethod())
            .notes(request.getNotes())
            .build();
        
        // 주문 상품 추가
        for (OrderItemRequest itemRequest : request.getOrderItems()) {
            addOrderItem(order, itemRequest);
        }
        
        order.calculateTotalAmount();
        
        Order savedOrder = orderRepository.save(order);
        log.info("주문이 생성되었습니다: {} (사용자: {})", savedOrder.getOrderNumber(), user.getUsername());
        
        return convertToResponse(savedOrder);
    }
    
    @Transactional
    public OrderResponse updateOrderStatus(Long orderId, OrderStatus newStatus) {
        Order order = findOrderById(orderId);
        
        validateStatusTransition(order.getStatus(), newStatus);
        
        order.updateStatus(newStatus);
        
        Order updatedOrder = orderRepository.save(order);
        log.info("주문 상태가 변경되었습니다: {} ({} -> {})", 
                 order.getOrderNumber(), order.getStatus(), newStatus);
        
        return convertToResponse(updatedOrder);
    }
    
    @Transactional
    public void cancelOrder(Long orderId) {
        Order order = findOrderById(orderId);
        
        if (!order.canBeCancelled()) {
            throw new BusinessException("취소할 수 없는 주문 상태입니다: " + order.getStatus());
        }
        
        // 재고 복원
        restoreStock(order);
        
        order.updateStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
        
        log.info("주문이 취소되었습니다: {}", order.getOrderNumber());
    }
    
    public BigDecimal calculateRevenueByDateRange(LocalDateTime startDate, LocalDateTime endDate) {
        BigDecimal revenue = orderRepository.getTotalRevenueByDateRange(startDate, endDate);
        return revenue != null ? revenue : BigDecimal.ZERO;
    }
    
    public List<OrderResponse> getOrdersByDateRange(LocalDateTime startDate, LocalDateTime endDate) {
        List<Order> orders = orderRepository.findByDateRange(startDate, endDate);
        return orders.stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }
    
    private Order findOrderById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("주문을 찾을 수 없습니다: " + id));
    }
    
    private User findUserById(Long userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("사용자를 찾을 수 없습니다: " + userId));
    }
    
    private Product findProductById(Long productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("상품을 찾을 수 없습니다: " + productId));
    }
    
    private void addOrderItem(Order order, OrderItemRequest itemRequest) {
        Product product = findProductById(itemRequest.getProductId());
        
        if (!product.isAvailable()) {
            throw new BusinessException("주문할 수 없는 상품입니다: " + product.getName());
        }
        
        if (product.getStockQuantity() < itemRequest.getQuantity()) {
            throw new BusinessException("재고가 부족합니다: " + product.getName());
        }
        
        // 재고 감소
        product.decreaseStock(itemRequest.getQuantity());
        productRepository.save(product);
        
        OrderItem orderItem = OrderItem.builder()
            .order(order)
            .product(product)
            .quantity(itemRequest.getQuantity())
            .unitPrice(product.getPrice())
            .build();
        
        order.addOrderItem(orderItem);
    }
    
    private void restoreStock(Order order) {
        for (OrderItem item : order.getOrderItems()) {
            Product product = item.getProduct();
            product.increaseStock(item.getQuantity());
            productRepository.save(product);
        }
    }
    
    private void validateStatusTransition(OrderStatus currentStatus, OrderStatus newStatus) {
        // 비즈니스 로직에 따른 상태 전환 유효성 검사
        switch (currentStatus) {
            case PENDING:
                if (newStatus != OrderStatus.CONFIRMED && newStatus != OrderStatus.CANCELLED) {
                    throw new BusinessException("유효하지 않은 상태 전환입니다");
                }
                break;
            case CONFIRMED:
                if (newStatus != OrderStatus.PROCESSING && newStatus != OrderStatus.CANCELLED) {
                    throw new BusinessException("유효하지 않은 상태 전환입니다");
                }
                break;
            case PROCESSING:
                if (newStatus != OrderStatus.SHIPPED && newStatus != OrderStatus.CANCELLED) {
                    throw new BusinessException("유효하지 않은 상태 전환입니다");
                }
                break;
            case SHIPPED:
                if (newStatus != OrderStatus.DELIVERED) {
                    throw new BusinessException("유효하지 않은 상태 전환입니다");
                }
                break;
            case DELIVERED:
                if (newStatus != OrderStatus.REFUNDED) {
                    throw new BusinessException("유효하지 않은 상태 전환입니다");
                }
                break;
            case CANCELLED:
            case REFUNDED:
                throw new BusinessException("더 이상 상태를 변경할 수 없습니다");
        }
    }
    
    private String generateOrderNumber() {
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
        String randomSuffix = String.valueOf((int) (Math.random() * 1000));
        return "ORD" + timestamp + randomSuffix;
    }
    
    private OrderResponse convertToResponse(Order order) {
        List<OrderItemResponse> orderItemResponses = order.getOrderItems().stream()
            .map(this::convertToOrderItemResponse)
            .collect(Collectors.toList());
        
        return OrderResponse.builder()
            .id(order.getId())
            .orderNumber(order.getOrderNumber())
            .userId(order.getUser().getId())
            .userFullName(order.getUser().getFullName())
            .status(order.getStatus())
            .totalAmount(order.getTotalAmount())
            .shippingAddress(order.getShippingAddress())
            .billingAddress(order.getBillingAddress())
            .paymentMethod(order.getPaymentMethod())
            .notes(order.getNotes())
            .shippedAt(order.getShippedAt())
            .deliveredAt(order.getDeliveredAt())
            .createdAt(order.getCreatedAt())
            .updatedAt(order.getUpdatedAt())
            .orderItems(orderItemResponses)
            .build();
    }
    
    private OrderItemResponse convertToOrderItemResponse(OrderItem orderItem) {
        return OrderItemResponse.builder()
            .id(orderItem.getId())
            .productId(orderItem.getProduct().getId())
            .productName(orderItem.getProduct().getName())
            .productSku(orderItem.getProduct().getSku())
            .quantity(orderItem.getQuantity())
            .unitPrice(orderItem.getUnitPrice())
            .subtotal(orderItem.getSubtotal())
            .build();
    }
}
```

## 🎮 나머지 Controller 클래스들

### ProductController

```java
// controller/ProductController.java
package com.example.app.controller;

import com.example.app.dto.request.CreateProductRequest;
import com.example.app.dto.request.UpdateProductRequest;
import com.example.app.dto.response.ApiResponse;
import com.example.app.dto.response.PageResponse;
import com.example.app.dto.response.ProductResponse;
import com.example.app.entity.ProductCategory;
import com.example.app.service.ProductService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
@Tag(name = "Product", description = "상품 관리 API")
public class ProductController {
    
    private final ProductService productService;
    
    @GetMapping
    @Operation(summary = "상품 목록 조회", description = "페이징된 상품 목록을 조회합니다")
    public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> getAllProducts(
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<ProductResponse> products = productService.getAllProducts(pageable);
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "상품 상세 조회", description = "ID로 상품 상세 정보를 조회합니다")
    public ResponseEntity<ApiResponse<ProductResponse>> getProductById(
            @Parameter(description = "상품 ID") @PathVariable Long id) {
        
        ProductResponse product = productService.getProductById(id);
        return ResponseEntity.ok(ApiResponse.success(product));
    }
    
    @GetMapping("/sku/{sku}")
    @Operation(summary = "SKU로 상품 조회", description = "SKU로 상품을 조회합니다")
    public ResponseEntity<ApiResponse<ProductResponse>> getProductBySku(
            @Parameter(description = "상품 SKU") @PathVariable String sku) {
        
        ProductResponse product = productService.getProductBySku(sku);
        return ResponseEntity.ok(ApiResponse.success(product));
    }
    
    @GetMapping("/category/{category}")
    @Operation(summary = "카테고리별 상품 조회", description = "특정 카테고리의 상품들을 조회합니다")
    public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> getProductsByCategory(
            @Parameter(description = "상품 카테고리") @PathVariable ProductCategory category,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<ProductResponse> products = productService.getProductsByCategory(category, pageable);
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @GetMapping("/search")
    @Operation(summary = "상품 검색", description = "키워드로 상품을 검색합니다")
    public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> searchProducts(
            @Parameter(description = "검색 키워드") @RequestParam String keyword,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<ProductResponse> products = productService.searchProducts(keyword, pageable);
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @GetMapping("/price-range")
    @Operation(summary = "가격 범위별 상품 조회", description = "가격 범위에 해당하는 상품들을 조회합니다")
    public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> getProductsByPriceRange(
            @Parameter(description = "최소 가격") @RequestParam BigDecimal minPrice,
            @Parameter(description = "최대 가격") @RequestParam BigDecimal maxPrice,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<ProductResponse> products = productService.getProductsByPriceRange(minPrice, maxPrice, pageable);
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @GetMapping("/low-stock")
    @Operation(summary = "재고 부족 상품 조회", description = "재고가 부족한 상품들을 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<List<ProductResponse>>> getLowStockProducts(
            @Parameter(description = "재고 임계값") @RequestParam(defaultValue = "10") Integer threshold) {
        
        List<ProductResponse> products = productService.getLowStockProducts(threshold);
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @PostMapping
    @Operation(summary = "상품 생성", description = "새로운 상품을 생성합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<ProductResponse>> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        
        ProductResponse createdProduct = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success("상품이 성공적으로 생성되었습니다", createdProduct));
    }
    
    @PutMapping("/{id}")
    @Operation(summary = "상품 정보 수정", description = "상품 정보를 수정합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<ProductResponse>> updateProduct(
            @Parameter(description = "상품 ID") @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        
        ProductResponse updatedProduct = productService.updateProduct(id, request);
        return ResponseEntity.ok(ApiResponse.success("상품 정보가 성공적으로 수정되었습니다", updatedProduct));
    }
    
    @DeleteMapping("/{id}")
    @Operation(summary = "상품 삭제", description = "상품을 삭제합니다")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<Void>> deleteProduct(
            @Parameter(description = "상품 ID") @PathVariable Long id) {
        
        productService.deleteProduct(id);
        return ResponseEntity.ok(ApiResponse.success("상품이 성공적으로 삭제되었습니다", null));
    }
    
    @PatchMapping("/{id}/stock")
    @Operation(summary = "재고 수량 변경", description = "상품의 재고 수량을 변경합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<Void>> updateStock(
            @Parameter(description = "상품 ID") @PathVariable Long id,
            @Parameter(description = "변경할 수량 (양수: 증가, 음수: 감소)") @RequestParam int quantity) {
        
        productService.updateStock(id, quantity);
        return ResponseEntity.ok(ApiResponse.success("재고가 성공적으로 업데이트되었습니다", null));
    }
}
```

### OrderController

```java
// controller/OrderController.java
package com.example.app.controller;

import com.example.app.dto.request.CreateOrderRequest;
import com.example.app.dto.response.ApiResponse;
import com.example.app.dto.response.OrderResponse;
import com.example.app.dto.response.PageResponse;
import com.example.app.entity.OrderStatus;
import com.example.app.security.CustomUserDetails;
import com.example.app.service.OrderService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
@Tag(name = "Order", description = "주문 관리 API")
public class OrderController {
    
    private final OrderService orderService;
    
    @GetMapping
    @Operation(summary = "주문 목록 조회", description = "페이징된 주문 목록을 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<PageResponse<OrderResponse>>> getAllOrders(
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<OrderResponse> orders = orderService.getAllOrders(pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "주문 상세 조회", description = "ID로 주문 상세 정보를 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER') or @orderService.getOrderById(#id).userId == authentication.principal.id")
    public ResponseEntity<ApiResponse<OrderResponse>> getOrderById(
            @Parameter(description = "주문 ID") @PathVariable Long id) {
        
        OrderResponse order = orderService.getOrderById(id);
        return ResponseEntity.ok(ApiResponse.success(order));
    }
    
    @GetMapping("/order-number/{orderNumber}")
    @Operation(summary = "주문번호로 주문 조회", description = "주문번호로 주문을 조회합니다")
    public ResponseEntity<ApiResponse<OrderResponse>> getOrderByOrderNumber(
            @Parameter(description = "주문번호") @PathVariable String orderNumber) {
        
        OrderResponse order = orderService.getOrderByOrderNumber(orderNumber);
        return ResponseEntity.ok(ApiResponse.success(order));
    }
    
    @GetMapping("/my-orders")
    @Operation(summary = "내 주문 목록 조회", description = "로그인한 사용자의 주문 목록을 조회합니다")
    public ResponseEntity<ApiResponse<PageResponse<OrderResponse>>> getMyOrders(
            @AuthenticationPrincipal CustomUserDetails userDetails,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<OrderResponse> orders = orderService.getOrdersByUser(userDetails.getId(), pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }
    
    @GetMapping("/user/{userId}")
    @Operation(summary = "사용자별 주문 조회", description = "특정 사용자의 주문 목록을 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<PageResponse<OrderResponse>>> getOrdersByUser(
            @Parameter(description = "사용자 ID") @PathVariable Long userId,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<OrderResponse> orders = orderService.getOrdersByUser(userId, pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }
    
    @GetMapping("/status/{status}")
    @Operation(summary = "상태별 주문 조회", description = "특정 상태의 주문들을 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<PageResponse<OrderResponse>>> getOrdersByStatus(
            @Parameter(description = "주문 상태") @PathVariable OrderStatus status,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<OrderResponse> orders = orderService.getOrdersByStatus(status, pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }
    
    @PostMapping
    @Operation(summary = "주문 생성", description = "새로운 주문을 생성합니다")
    public ResponseEntity<ApiResponse<OrderResponse>> createOrder(
            @AuthenticationPrincipal CustomUserDetails userDetails,
            @Valid @RequestBody CreateOrderRequest request) {
        
        OrderResponse createdOrder = orderService.createOrder(userDetails.getId(), request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success("주문이 성공적으로 생성되었습니다", createdOrder));
    }
    
    @PatchMapping("/{id}/status")
    @Operation(summary = "주문 상태 변경", description = "주문의 상태를 변경합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<OrderResponse>> updateOrderStatus(
            @Parameter(description = "주문 ID") @PathVariable Long id,
            @Parameter(description = "새로운 주문 상태") @RequestParam OrderStatus status) {
        
        OrderResponse updatedOrder = orderService.updateOrderStatus(id, status);
        return ResponseEntity.ok(ApiResponse.success("주문 상태가 성공적으로 변경되었습니다", updatedOrder));
    }
    
    @PostMapping("/{id}/cancel")
    @Operation(summary = "주문 취소", description = "주문을 취소합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER') or @orderService.getOrderById(#id).userId == authentication.principal.id")
    public ResponseEntity<ApiResponse<Void>> cancelOrder(
            @Parameter(description = "주문 ID") @PathVariable Long id) {
        
        orderService.cancelOrder(id);
        return ResponseEntity.ok(ApiResponse.success("주문이 성공적으로 취소되었습니다", null));
    }
    
    @GetMapping("/revenue")
    @Operation(summary = "매출 조회", description = "기간별 매출을 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<BigDecimal>> getRevenue(
            @Parameter(description = "시작 날짜") @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startDate,
            @Parameter(description = "종료 날짜") @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endDate) {
        
        BigDecimal revenue = orderService.calculateRevenueByDateRange(startDate, endDate);
        return ResponseEntity.ok(ApiResponse.success("매출 조회가 완료되었습니다", revenue));
    }
    
    @GetMapping("/date-range")
    @Operation(summary = "기간별 주문 조회", description = "특정 기간의 주문들을 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<ApiResponse<List<OrderResponse>>> getOrdersByDateRange(
            @Parameter(description = "시작 날짜") @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startDate,
            @Parameter(description = "종료 날짜") @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endDate) {
        
        List<OrderResponse> orders = orderService.getOrdersByDateRange(startDate, endDate);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }
}
```

## 🧪 완전한 테스트 코드

### 통합 테스트

```java
// test/java/com/example/app/controller/UserControllerIntegrationTest.java
package com.example.app.controller;

import com.example.app.dto.request.CreateUserRequest;
import com.example.app.entity.User;
import com.example.app.entity.UserRole;
import com.example.app.entity.UserStatus;
import com.example.app.repository.UserRepository;
import com.example.app.security.JwtTokenProvider;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureWebMvcSecurity;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.context.WebApplicationContext;

import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
@Transactional
@DisplayName("사용자 컨트롤러 통합 테스트")
class UserControllerIntegrationTest {
    
    @Autowired
    private WebApplicationContext context;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    private MockMvc mockMvc;
    private String adminToken;
    private User testUser;
    
    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders
            .webAppContextSetup(context)
            .apply(springSecurity())
            .build();
        
        // 테스트용 관리자 사용자 생성
        User admin = User.builder()
            .username("admin")
            .email("admin@example.com")
            .password(passwordEncoder.encode("password"))
            .firstName("Admin")
            .lastName("User")
            .role(UserRole.ADMIN)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .build();
        userRepository.save(admin);
        
        adminToken = tokenProvider.generateToken("admin");
        
        // 테스트용 일반 사용자 생성
        testUser = User.builder()
            .username("testuser")
            .email("test@example.com")
            .password(passwordEncoder.encode("password"))
            .firstName("Test")
            .lastName("User")
            .role(UserRole.USER)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .build();
        userRepository.save(testUser);
    }
    
    @Test
    @DisplayName("사용자 목록 조회 성공")
    void getAllUsers_Success() throws Exception {
        mockMvc.perform(get("/api/users")
                .header("Authorization", "Bearer " + adminToken)
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.content").isArray());
    }
    
    @Test
    @DisplayName("사용자 생성 성공")
    void createUser_Success() throws Exception {
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("newuser");
        request.setEmail("newuser@example.com");
        request.setPassword("password123");
        request.setFirstName("New");
        request.setLastName("User");
        request.setRole(UserRole.USER);
        
        mockMvc.perform(post("/api/users")
                .header("Authorization", "Bearer " + adminToken)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpected(status().isCreated())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.username").value("newuser"));
    }
    
    @Test
    @DisplayName("권한 없는 접근 실패")
    void unauthorized_Access() throws Exception {
        mockMvc.perform(get("/api/users")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isUnauthorized());
    }
}

// test/java/com/example/app/ApplicationTests.java
package com.example.app;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class ApplicationTests {
    
    @Test
    void contextLoads() {
    }
}
```

## 🐳 Docker 설정

### Dockerfile

```dockerfile
# Dockerfile
FROM openjdk:17-jdk-slim as builder

WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .
COPY src src

RUN chmod +x ./gradlew
RUN ./gradlew bootJar

FROM openjdk:17-jre-slim

WORKDIR /app

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=jdbc:postgresql://postgres:5432/appdb
      - DATABASE_USERNAME=appuser
      - DATABASE_PASSWORD=apppassword
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - JWT_SECRET=your-secret-key-here-change-this-in-production
    depends_on:
      - postgres
      - redis
    networks:
      - app-network
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

## 📋 추가 DTO 클래스들

### 요청 DTO 추가

```java
// dto/request/UpdateUserRequest.java
package com.example.app.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class UpdateUserRequest {
    
    @Email(message = "올바른 이메일 형식이 아닙니다")
    @Size(max = 100, message = "이메일은 100자를 초과할 수 없습니다")
    private String email;
    
    @Size(max = 50, message = "이름은 50자를 초과할 수 없습니다")
    private String firstName;
    
    @Size(max = 50, message = "성은 50자를 초과할 수 없습니다")
    private String lastName;
    
    @Pattern(regexp = "^[0-9-+() ]*$", message = "올바른 전화번호 형식이 아닙니다")
    private String phone;
}

// dto/request/UpdateProductRequest.java
package com.example.app.dto.request;

import com.example.app.entity.ProductCategory;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Digits;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class UpdateProductRequest {
    
    @Size(max = 200, message = "상품명은 200자를 초과할 수 없습니다")
    private String name;
    
    @Size(max = 1000, message = "상품 설명은 1000자를 초과할 수 없습니다")
    private String description;
    
    @DecimalMin(value = "0.01", message = "가격은 0.01 이상이어야 합니다")
    @Digits(integer = 17, fraction = 2, message = "가격 형식이 올바르지 않습니다")
    private BigDecimal price;
    
    @Min(value = 0, message = "재고 수량은 0 이상이어야 합니다")
    private Integer stockQuantity;
    
    @Size(max = 100, message = "SKU는 100자를 초과할 수 없습니다")
    private String sku;
    
    private ProductCategory category;
    
    @Pattern(regexp = "^https?://.*", message = "올바른 URL 형식이 아닙니다")
    private String imageUrl;
    
    @DecimalMin(value = "0.01", message = "무게는 0.01 이상이어야 합니다")
    private BigDecimal weight;
    
    private String dimensions;
}
```

이렇게 Spring Boot 보일러플레이트 코드가 완성되었습니다! 메인 파일에서 누락되었던 부분들을 모두 포함하여 실무에서 바로 사용할 수 있는 완전한 구조를 제공합니다.
