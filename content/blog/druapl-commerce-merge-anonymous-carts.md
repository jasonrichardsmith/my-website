+++
title = "Drupal Commerce merge anonymous carts when logging in"
+++

So I was recently asked to take a cart for an anonymous user and merge it with their existing cart when they logged in, if they had added items to their cart prior to logging in.

Unbeknownst to me was the fact that Drupal would clobber an existing cart on an account with the anonymous cart once a user logged in.  Basically it appears that Drupal switches the anonymous cart to the user.

I have not tested it but from the comments it sounds like once the user checkouts with the cart from the anonymous user it is replaced with their original cart.

So Google revealed this post, which pushed me towards hook_commerce_cart_order_convert.

Your use case may require more logic.  In this situation, every user could only order one of each product.  How you may want to handle quantity is up to you.

So here is how I did it:
```php
//$order_wrapper is the cart from the anonymous user that is going to replace the users cart, $account is the user log in.
function foo_commerce_cart_order_convert($order_wrapper, $account){
  // This loads the cart assigned with the user
  $original_cart = commerce_cart_order_load($account-\u003euid);
  //Does a cart exist for the logged in user, because it will be overridden with the new one
  if($original_cart){
    //Wrap it
    $account_order = entity_metadata_wrapper('commerce_order', $orginal_cart);
    $existing_prods = array();
    // Get existing products.
    foreach($order_wrapper-\u003ecommerce_line_items as $line_item){
      $existing_prods[] = $line_item-\u003ecommerce_product-\u003eproduct_id-\u003evalue();
    }
    foreach($account_order-\u003ecommerce_line_items as $line_item){
      // If not in array, the anonymous cart didn have the product
      if(!in_array($line_item-\u003ecommerce_product-\u003eproduct_id-\u003evalue(), $existing_prods)){
         // change the order id for the line item
        $line_item-\u003eorder_id = $order_wrapper-\u003eorder_id-\u003evalue();
        // Not sure if this was needed but it didnt hurt
        $line_item-\u003esave();
        // add the line item to the new order
        $order_wrapper-\u003ecommerce_line_items[] = $line_item;
      }   
    }
  //  We are clearing out the users previous cart since it was replaced with the anonymous cart
  // Important!! If this is not done, and you delete an item from your cart after checkout the item diappears from your previous checkout.
  // Basically you have an order and a cart with the same line item id. 
  $account_order-\u003ecommerce_line_items = array();
  $account_order-\u003esave();   
  }
}
```
