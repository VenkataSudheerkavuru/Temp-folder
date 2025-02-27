<div *ngIf="selectedContact" class="card mt-3 shadow-sm" style="width: 24rem;">
  <div class="card-body">
    <h5 class="card-title">{{ selectedContact.name }}</h5>
    <p class="card-text">
      <strong>Email:</strong> {{ selectedContact.email }}<br>
      <strong>Mobile:</strong> {{ selectedContact.mobile }}<br>
      <span *ngIf="selectedContact.landline">
        <strong>Landline:</strong> {{ selectedContact.landline }}<br>
      </span>
      <span *ngIf="selectedContact.website">
        <strong>Website:</strong>
        <a [href]="selectedContact.website" target="_blank">{{ selectedContact.website }}</a><br>
      </span>
      <strong>Address:</strong><br>
      {{ selectedContact.address }}
    </p>
    <div class="d-flex justify-content-end">
      <button class="btn btn-primary btn-sm me-2" (click)="editContact(selectedContact)">
        Edit
      </button>
      <button class="btn btn-danger btn-sm" (click)="deleteContact(selectedContact)">
        Delete
      </button>
    </div>
  </div>
</div>
